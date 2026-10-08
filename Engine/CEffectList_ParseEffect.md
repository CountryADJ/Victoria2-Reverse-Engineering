# Victoria 2 effect system: `CEffect::ParseEffect`, effect vtables, `assimilate`, `add_accepted_culture` and the `NameMap_FindByName` family

Reverse-engineering notes for the effect factory of Victoria 2's script parser, the effect class layout (vtable), two concrete effects (`assimilate`, `add_accepted_culture`) and the name-lookup helpers they use.

> Confidence: structure, calling conventions, offsets, list/vtable behavior and hash-map mechanics are read directly from the decompilation/disassembly. Class and keyword names are inferred. Anything marked *(inferred)* is not confirmed.

---

## 0. Conventions and glossary

### `ESI` is the country

Several game functions are `__usercall` and take the **country object in `ESI`** (`Country*`). Wherever this document says "`ESI = country`", it means the register holds a pointer to the country struct of the current scope. It is a runtime pointer, so it has no fixed value. It is computed like this (see `FUN_00893490`, `0x893525` and `0x89354B`..`0x893555`):

```
mgr   = [DAT_012587E4]        ; countries manager            (RVA 0xE587E4)
begin = [mgr + 4]             ; start of the countries vector
idx   = [ctx + 0x0C]          ; country index of the current scope
ESI   = [begin + idx * 4]     ; Country*
```

To observe it live, break at `0x522310` (`Country_AddAcceptedCulture`) and read `ESI`.

### Globals

| VA | RVA | Meaning |
|---|---|---|
| `DAT_012588E8` | `0xE588E8` | world/session pointer (`+0xACC` = province array, `+0xB60` = player country index) |
| `DAT_012587E4` | `0xE587E4` | countries manager (`+4` = vector begin) |
| `DAT_012587E8` | `0xE587E8` | culture map (name → culture), used by `CultureMap_FindByName` |
| `DAT_012587EC` | `0xE587EC` | **null/default culture** singleton (`0x98` bytes) *(inferred)* |
| `DAT_0131BB34` | `0xF1BB34` | pop type database (`+0xC`/`+0x10` = vector of pop types, created lazily) |

### Generic list structures (used everywhere in the engine)

```c
struct ListNode { void* value; ListNode* prev; ListNode* next; uint8_t flag; };  // 0x10 bytes
struct List     { ListNode* head; ListNode* tail; int count; };
```

Append: allocate a node `{value, prev = tail, next = 0}`, `tail = node`, `count++`; if `count` was 0 then `head = node`, else `oldTail->next = node`. Effects keep their child list at `+0x08/+0x0C/+0x10`; the country keeps its accepted-culture list at `+0xE54/+0xE58/+0xE5C`.

### Known struct fields

| Struct | Offset | Meaning |
|---|---|---|
| Country | `+0xE54` / `+0xE58` / `+0xE5C` | accepted cultures list: head / tail / count |
| Country | `+0xE64` | primary culture |
| Province | `+0x12C` | owner (country index) |
| Province | `+0x194` | pop lists, stride `0x10` per pop type `{head, tail, count, ?}` |
| Pop | `+0x6C` | culture |
| Pop | `+0x27C` | next pop in list |
| Culture | `+0x18` | name (`std::string`) |
| Culture | `+0x50` | pointer to the culture group *(inferred)*; the group's member list head is at `group + 8` *(inferred)* |
| Parser ctx (effect `ctx`) | `+0x0C` / `+0x10` / `+0x14` | country index / province id / pop |

---

## 1. Effect class layout (vtable)

Every effect object starts with a common base:

```c
e[0] = vtable
e[1] = 0x18D        // base marker/type (inferred)
e[2] = head   e[3] = tail   e[4] = count     // child effect list (+0x08/+0x0C/+0x10)
e[5] (byte) = 0     // +0x14
e[6] = keyId        // +0x18, set by ParseEffect
e[7] (word) = 0     // +0x1C
// subclass fields start at +0x20
```

Because every effect embeds a child list and **vtable slot 4 is `FUN_00876B30`** (confirmed on two different effects), the `this` of `ParseEffect` is **the effect/block currently being parsed**, not a separate list object. This is why blocks like `any_pop = { ... }` can nest effects.

### Comparison of two vtables

| Slot | Offset | `add_accepted_culture` (`DAT_00E2B9C8`) | `assimilate` (`DAT_00E2BAE8`) | Meaning |
|---|---|---|---|---|
| 0 | `+0x00` | `0x87A3F0` | `0x87A3F0` | destructor *(inferred)* |
| 1 | `+0x04` | `0x9A29F0` | `0x9A29F0` | shared base method |
| 2 | `+0x08` | `0xA46A60` | `0xA46A60` | shared base method |
| 3 | `+0x0C` | `0x9A2A90` | `0x9A2A90` | `ParseBlock` (base default; block-capable effects override it) |
| **4** | `+0x10` | **`0x876B30`** | **`0x876B30`** | **`ParseEffect`** (this function) |
| 5 | `+0x14` | `0x4010C0` | `0x4010C0` | tiny stub |
| 6 | `+0x18` | `0x45D720` | `0x45D720` | shared |
| **7** | `+0x1C` | `0x893390` | `0x87C410` | **`ParseValue`** (token by value) |
| 8 | `+0x20` | `0x87A6C0` | `0x87A6C0` | shared |
| **9** | `+0x24` | `0x893570` | `0x894260` | tooltip/description *(inferred)*; sits right after `Execute` in both |
| 10 | `+0x28` | `0x928300` | `0x928300` | shared |
| **11** | `+0x2C` | `0x893490` | `0x894160` | **`Execute`** |
| 12 | `+0x30` | `0xA31490` | `0xA31490` | shared |
| 13 | `+0x34` | (data, not code) | `0x87C450` | only in the bool-style class |

Notes:

- `assimilate`'s `ParseValue` (`0x87C410`) and slot 13 (`0x87C450`) are `0x40` apart; they probably belong to a shared "bool effect" base (`= yes`) *(inferred)*, with the bool at `+0x20`.
- In both effects the code order is `ParseValue` → `Execute` → slot 9.
- The word after the last slot of `add_accepted_culture` (`0xE5ED18`) points to data, so its vtable ends at slot 12.

---

## 2. `CEffect::ParseEffect` (FUN_00876B30), vtable slot 4

Effect factory. Given a keyword ID read from an effect block, it allocates and constructs the matching effect object, appends it to the current effect's child list, and makes the new effect parse its own value.

| | |
|---|---|
| **Ghidra name** | `FUN_00876B30` |
| **Proposed name** | `CEffect::ParseEffect` (alt: `CEffect::ParseChildEffect`) |
| **RVA** | `0x476B30` (VA `0x876B30`) |
| **Convention** | `__thiscall`, `ret 8` |
| **Prologue** | `55 8B EC 83 E4 F8` (`push ebp; mov ebp,esp; and esp,-8`) |
| **Returns** | nothing; the result is the new effect appended to the list |

### Signature

```c
void __thiscall CEffect::ParseEffect(
    CEffect*   this,      // param_1: the effect/block being parsed (child list at +8/+0xC/+0x10)
    ParserCtx* ctx,       // param_2: script parser / lexer context
    int        keyId);    // param_3: keyword ID of the effect
```

### `ctx` (parser context), fields used here

| Offset | Meaning |
|---|---|
| `+0x1C` | stream/lexer object (used for error output) |
| `+0x24` | location info used in the error message (file/line) *(inferred)* |
| `+0x228` | **current value token**: type at `+0x228`, `3` = `{` (open brace) |
| `+0x22C` | token text (value string) |

The token structure starting at `+0x228` is `0x41` dwords (260 bytes). It is copied **by value** onto the stack when passed to a virtual method.

### Behavior

1. **Select and construct the effect.** A large `switch` on `keyId` (about 110 cases, compiled as a binary split tree) calls `operator_new(size)` and initializes the object, inline or through a constructor `FUN_008xxxxx`. Size and vtable differ per keyword.
2. **Append to the child list** as described in section 0.
3. **Let the effect read its value.** Two paths:

   | Path | Call | Used by |
   |---|---|---|
   | **Block-capable** | `effect->vtable[3]` (`+0x0C`) with `ctx` | Effects with their own parser. They must check for `{` themselves. Example: `0x3FF` checks `ctx[0x228] == 3` first. |
   | **Simple value** | `effect->vtable[7]` (`+0x1C`) with a by-value copy of the token (`ctx+0x228`, 0x41 dwords) | Effects that take a single value token. **No `{` check happens here.** |

4. **Store the keyword ID**: `if (effect) effect[6] = keyId;` (offset `+0x18`).

### Why `assimilate = { ... }` is not supported by vanilla

`keyId 0xA5B` allocates `0x24` bytes, sets vtable `0xE2BAE8` and falls into the **simple value** path. The token is `{`, nothing validates it, and the inner keys are then read as keys of the parent effect.

### Selected cases

All use the shared base initialization and the **simple value** path. Keyword names are not resolved except where noted.

| keyId | Size | Vtable (VA) | Note |
|---|---|---|---|
| `0x1CB` | `0x24` | `0xE2C4E8` | |
| `0x28F` | `0x28` | `0xE2B9C8` | **`add_accepted_culture`** *(inferred from behavior)* |
| `0x290` | `0x24` | `0xE2BA38` | |
| `0x292` | `0x24` | `0xE2BB60` | |
| `0x29C` | `0x24` | `0xE2B958` | |
| `0x2A2` | `0x24` | `0xE2A8EC` | |
| `0x2BF` | `0x28` | `0xE2A99C` | |
| `0x641` | `0x24` | `0xE2B920` | |
| `0x865` | `0x24` | `0xE2B990` | |
| `0x869` | `0x24` | `0xE2BA70` | |
| `0x89C` | `0x24` | `0xE2BD2C` | |
| `0x9FE` | `0x24` | `0xE2B720` | |
| **`0xA5B`** | `0x24` | **`0xE2BAE8`** | **`assimilate`** |

Effects with their own constructors and the **block-capable** path (not exhaustive): `0x343`, `0x452`, `0x8AD`, `0x8AE` (all `FUN_0089C770` family, size `0x4C`), `0x3FF`, `0x403`, `0x988`, `0x98F`.

The keyword string table at `DAT_00E22D44` has not been decoded yet (only its first byte, `0x74` = `'t'`, was seen). To map `keyId → name`, search for the string `add_accepted_culture` and look for the `0x28F` constant next to its XREF.

### Default case (unknown keyword)

If `keyId` matches no case (`switchD_00876b87_caseD_1c0`), the identifier text (`ctx+0x22C`) is looked up by name in several game tables, in this order. The first hit creates a different effect type *(inferred: shorthand/scope effects such as `TAG = { }`, `<province id> = { }`, `<culture> = ...`)*.

| # | Lookup | Extra check | Effect created (size) |
|---|---|---|---|
| 1 | `NameMap_FindByName_Key20` (`FUN_004DC420`); on miss, `FUN_004DACC0` (probably same family) | `vtable[0x24]` accepts it | `FUN_00891C70` (`0x28`) |
| 2 | `NameMap_FindByName_Key2AC` (`FUN_0057B0A0`) | `vtable[0x20]` accepts it | `FUN_008806E0` (`0x28`) |
| 3 | `NameMap_FindByName_Key10` (`FUN_005677B0`) | none | `FUN_0087DF90` (`0x28`) |
| 4 | Pop type DB `DAT_0131BB34` via `FUN_0096DC20` | `vtable[0x18]` accepts it | `FUN_0089B4B0` (`0x40`) |
| 5 | `CultureMap_FindByName` (`FUN_004A2F70`) | none | `FUN_00888B50` (`0x40`) |
| 6 | `NameMap_FindByName_Key0C` (`FUN_00973BD0`) | none | `FUN_008982F0` (`0x28`) |
| 7 | nothing matched | see below | see below |

Nothing matched:

- If the value token is **not** `{`: write an error to the log stream (using `ctx+0x24` as location) and return without adding an effect.
- If it is `{`: create a generic `0x24`-byte object via `FUN_0087C9A0` (token copied by value), add it, then call `vtable[7]`.

Unidentified helpers called along the way: `FUN_004DAD40`, `FUN_00576F80`, `FUN_004EBCB0`. They run before the lookups and probably prepare the key string or select the map *(inferred)*. The map is passed in `EAX`, which the decompiler hides.

---

## 3. `assimilate` (keyId `0xA5B`, vtable `0xE2BAE8`)

### Vanilla `Execute` = `FUN_00894160` (slot 11)

`__thiscall`, `ret 8`, first stack argument is `ctx`.

1. Return if `[this + 0x20] == 0` (the `= yes` bool).
2. Province = `[ [DAT_012588E8] + 0xACC ][ ctx + 0x10 ]`.
3. Pop type DB `DAT_0131BB34` (created lazily with `FUN_0041A160` + `FUN_00423700`).
4. For every pop type, walk the province's pop list (`[prov + 0x194] + t*0x10`), following `pop + 0x27C`, and set `pop + 0x6C = [ countries[ [prov + 0x12C] ] + 0xE64 ]` (the **owner's primary culture**).

Vanilla behavior worth knowing:

- It **ignores the pop scope**: it always works on the whole province (`ctx + 0x10`).
- It does **not merge pops** and does not refresh the accepted flags.
- It reads `[prov + 0x128]` into a local that is never used.

All offsets match the mod's constants (`0x12C`, `0x194`, stride 16, `0x27C`, `0x6C`, `0xE64`, `0x20`). The mod's `amount = 1` branch additionally calls `AssimRefreshAccepted`, so it is slightly different from vanilla.

Suggested names: `FUN_0041A160` = `PopTypeDB_Construct`, `FUN_00423700` = `PopTypeDB_SetGlobal` *(inferred)*.

---

## 4. `add_accepted_culture` (keyId `0x28F`, vtable `0xE2B9C8`, size `0x28`)

Identified from behavior (the keyword string was not verified in the table). Fields: `+0x20` = culture, `+0x24` = `union` flag.

### `ParseValue` = `FUN_00893390` (slot 7)

`__thiscall`, **`ret 0x104`** (token passed by value, `0x41` dwords). The first stack dword is the token type; the token text starts at `[EBP+0xC]` (= `ctx+0x22C`).

1. `this->+0x24 = 0`.
2. Get the null-culture singleton (`DAT_012587EC`), creating it if needed (inline copy of `Culture_GetNullInstance`).
3. `this->+0x20 = singleton`.
4. If the token type is **`0x335`** (the `union` keyword token): `this->+0x24 = 1` and return.
5. Otherwise copy the token text into a `std::string`, look it up with `CultureMap_FindByName` (`FUN_004A2F70`, map `DAT_012587E8`). The first call is only an existence test; if it succeeds the lookup is repeated to get the result.
6. If the culture does not exist, the result is `Culture_GetNullInstance()` (`FUN_00561180`).
7. `this->+0x20 = result`.

So `union` is detected by **token type** (`0x335`), not by comparing text.

### `Execute` = `FUN_00893490` (slot 11)

`__thiscall`, `ret 8`, argument `ctx`. `country = countries[ctx + 0x0C]` (this is the `ESI` described in section 0).

- **`+0x24 == 0`**: `Country_AddAcceptedCulture(country, [this + 0x20])`.
- **`+0x24 != 0` (union)**: iterate the cultures of the primary culture's group (list at `[ [primary + 0x50] + 8 ]`). For each culture that is not the primary one and not already in `[country + 0xE54]`, call `Country_AddAcceptedCulture(country, culture)`. The "already accepted" test is redundant because the callee does it too.

### `Country_AddAcceptedCulture` = `FUN_00522310`

| | |
|---|---|
| **Convention** | usercall: **`ESI = Country*`**, one stack argument (`Culture*`), `ret 4` |
| **RVA** | `0x122310` (VA `0x522310`) |

1. Walk the accepted list from `[ESI + 0xE54]`; if the culture is already there, return.
2. Otherwise allocate a `ListNode {culture, prev = tail, next = 0, flag = 0}` and append it (`tail` at `+0xE58`, `count` at `+0xE5C`, `head` at `+0xE54` when the list was empty).
3. It does not touch pops; pop-level accepted flags are refreshed elsewhere.

Calling it from the mod:

```cpp
static void AddAcceptedCulture(void* country, void* culture)
{
    DWORD fn = g_base + 0x122310;   // VA 0x522310
    __asm
    {
        push esi
        mov  esi, country           // ESI = Country*
        push culture
        call fn                     // ret 4 pops the argument
        pop  esi
    }
}
```

### Null/default culture = `FUN_00561180` and `FUN_00561080`

`FUN_00561180` is a lazy singleton getter (no arguments, no lock):

```c
Obj* Culture_GetNullInstance() {
    if (DAT_012587EC == NULL) {
        Obj* p = operator_new(0x98);
        DAT_012587EC = p ? Culture_ConstructNull(p) : NULL;   // FUN_00561080
    }
    return DAT_012587EC;
}
```

It has 20+ callers. Because `ParseValue` stores it in a field that holds a culture, it is very likely a **null/default culture** object *(inferred)*.

Consequence: a misspelled culture name does not raise an error. The effect silently stores the null culture, and `Execute` then **adds the null culture to the country's accepted list**.

To confirm it is a culture: check `+0x18` (name) and `+0x50` (group) of the singleton.

---

## 5. The `NameMap_FindByName` family

`FUN_004DC420`, `FUN_0057B0A0`, `FUN_005677B0`, `FUN_00973BD0` and `FUN_004A2F70` are **the same function**: a lookup by name in a hash map, instantiated for different object types. Only the offset of the name inside the object differs.

### How they work

- **Convention:** `EAX = map`, one stack argument (`std::string* name`), `ret 4`. Returns the found object in `EAX`, or 0.
- **Hash:** `h = (h + byte) * 0x3D` for every byte of the name, then `h % bucketCount`. Case-sensitive.
- **Map layout:** `+4` bucket count, `+8` bucket array. Each node is `{object*, next}`.
- **Comparison:** `memcmp` over the shorter length plus length equality. The key is read from the object at `obj + OFFSET` (MSVC 2010 `std::string`: buffer/pointer, size at `+0x10`, capacity at `+0x14`).

Suggested Ghidra type: `void* __usercall f@<EAX>(NameMap* map@<EAX>, std::string* name)`, with `ret 4`.

### Proposed names

| Function | Proposed name | Name offset in the object |
|---|---|---|
| `FUN_004DC420` (0x4DC420) | `NameMap_FindByName_Key20` | `+0x20` |
| `FUN_0057B0A0` (0x57B0A0) | `NameMap_FindByName_Key2AC` | `+0x2AC` |
| `FUN_005677B0` (0x5677B0) | `NameMap_FindByName_Key10` | `+0x10` |
| `FUN_00973BD0` (0x973BD0) | `NameMap_FindByName_Key0C` | `+0x0C` |
| `FUN_004A2F70` (0x4A2F70) | `CultureMap_FindByName` | `+0x18` |

`FUN_004A2F70` is confirmed as the culture lookup (the mod's `AssimLookupCulture` and `add_accepted_culture`'s `ParseValue` both call it with `DAT_012587E8`). For the other four, the object type cannot be stated with certainty because the map comes in through `EAX` and the decompiler hides it. The `+0x2AC` offset means a very large struct; an unconfirmed suspicion is the country object with its tag as the name.

### How to find the real names

1. In the `FUN_00876B30` listing, go to the call sites `0x87895A`, `0x878A25`, `0x878A6A`, `0x878B8A` and `0x878BB7`.
2. Note which global is loaded into `EAX` right before each `CALL`.
3. That global is the map. If it is the culture map (`0xE587E8`) or a known tag/region map, rename accordingly.
4. `FUN_004DACC0`, the fallback of `FUN_004DC420`, is probably another instance of the same family.

---

## 6. Summary of proposed names

| Address | Ghidra name | Proposed name |
|---|---|---|
| `0x876B30` | `FUN_00876B30` | `CEffect::ParseEffect` |
| `0x894160` | `FUN_00894160` | `CAssimilateEffect::Execute` |
| `0x87C410` | `FUN_0087C410` | `CBoolEffect::ParseValue` *(inferred)* |
| `0x893390` | `FUN_00893390` | `CAddAcceptedCultureEffect::ParseValue` |
| `0x893490` | `FUN_00893490` | `CAddAcceptedCultureEffect::Execute` |
| `0xE2B9C8` | `DAT_00E2B9C8` | `vtbl_CAddAcceptedCultureEffect` |
| `0xE2BAE8` | `DAT_00E2BAE8` | `vtbl_CAssimilateEffect` |
| `0x522310` | `FUN_00522310` | `Country_AddAcceptedCulture` (usercall, `ESI = Country*`) |
| `0x561180` | `FUN_00561180` | `Culture_GetNullInstance` |
| `0x561080` | `FUN_00561080` | `Culture_ConstructNull` |
| `0x12587EC` | `DAT_012587EC` | `g_nullCulture` |
| `0x41A160` | `FUN_0041A160` | `PopTypeDB_Construct` *(inferred)* |
| `0x423700` | `FUN_00423700` | `PopTypeDB_SetGlobal` *(inferred)* |
| see section 5 | `FUN_004DC420`, `FUN_0057B0A0`, `FUN_005677B0`, `FUN_00973BD0`, `FUN_004A2F70` | `NameMap_FindByName_*` / `CultureMap_FindByName` |

---

## 7. Notes for hooking (as used by `ENABLE_ASSIMILATE_BLOCK`)

- The new effect is already linked at the **tail** of the parent's child list when `ParseEffect` returns: `node = *(this + 0xC); effect = *(void**)node;`.
- Check the class by comparing `*(DWORD*)effect` with the expected vtable before touching it.
- For `0xA5B`, `vtable[7]` has already consumed **one token** (the `{`) by the time the detour regains control, so a hook must resume reading from the lexer at that point and consume the closing `}` itself.
- Effect addresses can be recycled between loads, so any side table keyed by effect pointer must invalidate stale entries when a new effect is created at the same address.
- Stolen bytes: the prologue is 6 bytes (`55 8B EC 83 E4 F8`), safe to relocate into a trampoline.
- `Execute` is vtable slot 11 for all effects (`ret 8`, `ctx` as first stack argument), which is how the mod replaces `CAssimilateEffect::Execute`.
- To add accepted cultures from the mod, call `Country_AddAcceptedCulture` as shown in section 4. Unlike vanilla's parser, the mod's lookup can return 0 for an unknown culture instead of the null culture.

---

## 8. Open questions

- Resolve keyword names for every `keyId` (keyword table `DAT_00E22D44` not decoded yet) and confirm `0x28F` = `add_accepted_culture`.
- Identify the base class (`+0x04 = 0x18D`), the shared "bool effect" base (`0x87C410` / `0x87C450`) and the purpose of slots 1, 2, 5, 6, 8, 10, 12.
- Confirm slot 9 (tooltip/description) and slot 3 (`ParseBlock`) semantics.
- Identify which map each `NameMap_FindByName_*` instance receives, and what the effects built by `FUN_00891C70`, `FUN_008806E0`, `FUN_0087DF90`, `FUN_0089B4B0`, `FUN_00888B50` and `FUN_008982F0` do.
- Identify `FUN_004DAD40`, `FUN_00576F80` and `FUN_004EBCB0`.
- Verify that the `DAT_012587EC` object is a culture (check its `+0x18` name and `+0x50` group).
