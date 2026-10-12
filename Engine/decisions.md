# Decisions

Where and how often `potential` / `allow` are evaluated, and what the AI does with the result.

Source: research of October 2026. Keyword ids: [data/decision-keyword-map.txt](data/decision-keyword-map.txt).
Marks: **[C]** read in the code, **[M]** measured (TGC, 1836, max speed).

## Summary

- `CDecision` (vtable `0x00E29B54`, ctor `0x008743C0`, keyword handler `0x00874CE0`: `potential` token
  `0x729`, `allow` `0x72A`): name `+0x08` (std::string), `potential` `+0x24`, `allow` `+0x40` (both
  embedded `CAndTrigger`), `effect` `+0x5C`, `ai_will_do` `+0x110` (an MTTH-style block). **[C]**
- The manager `g_decisionManager` (`0x01319730`, 0x38 bytes, getter `DecisionManager_Get` `0x00875040`) has
  two `vector<CDecision*>` at `+0x0C` and `+0x28`. **[C]**
- The daily AI loop is `AI_DecisionUpdate` (`0x00821190`), once per AI country per day, driven by the
  per-frame AI job (see [../AI/Ai_daily_jobs.md](../AI/Ai_daily_jobs.md)). In TGC: 1651 decisions × ~320
  countries ≈ 529 k `potential` evaluations a day, ~23.5 ms of a ~200 ms day, ~2500 passes/day. **[M]**

## `AI_DecisionUpdate` (0x00821190) **[C]**

`__fastcall` with the AI object in `ECX` (`+0x30` tag, `+0x34` country index), called at `0x0081EA9B` from
`AI_CountryFrameJob`.

1. Scope: `CEventScope::ctor(scope, 0x2D)` (`PUSH 0x2D` at `0x008211B3`), then `scope+0x08/+0x0C` =
   AI object `+0x30/+0x34`. The same scope is passed to every `potential` and `allow`.
2. List B (`+0x28`) first, then list A (`+0x0C`). For each decision, in order:
   - `CDecision` vtable slot 6, called twice. In 3.04 it is `MOV AL,1 / RET` (`0x00A31490`, shared with
     `CCasusBelliType::IsRealType`): always true.
   - `potential.Evaluate(scope)` (`decision+0x24`), then `allow.Evaluate(scope)` (`decision+0x40`).
   - `CDecision_EvaluateAiWillDo` (`0x00874EA0`) -> chance; `rand() % 100` (the **CRT** `rand`,
     `0x00AB047B`, not the Mersenne Twister) against chance × 100.
   - On success: a 0xA0-byte command (`FUN_00905B30`, with a copy of the scope and the decision name) is
     handed to the session (`world+0xB24` vt `+0x44`, then `+0x18`), and the function returns.
3. **The decision is not executed here.** It runs later, when `CommandQueue_Process` executes the
   command. So while the AI queue is drained, the decision loop itself does not change the game.

## Other evaluation sites **[C]**

| site | when | what |
|---|---|---|
| `AI_DiplomacyUpdate` `0x0085D270` | a few countries a day (1–6 measured) | both lists through `DecisionManager_Get` (calls at `0x0085D34F` / `0x0085D459`, `ESI` = AI object, `+0x0C` index) |
| `Player_DecisionAlert` `0x005399C0` | player only (`0x0050B792`, guarded at `0x0050B780`), also per frame from `CInGameIdler_Update` (`0x00654E48`) | `potential && allow`, stops at the first available one |
| `CPoliticsView` (`0x0046B3A0`, `0x0047D0F0`), `0x006DB2E0`, `0x006D2DD0`, tooltip `0x0053A6D0` | on UI refresh | player |

## Script patterns (static count, TGC)

1653 decisions: 767 use "take once" (`set_country_flag` in the effect, `NOT has_country_flag` in
`potential`) with a flag nothing ever clears; 404 have a `NOT has_global_flag` gate on a global flag that
is set and never cleared; 666 are tag-locked (a top-level `tag = X`), which alone means ~213 k of the
daily evaluations are for the wrong country. First condition of `potential`: `tag` 637, `owns` 221,
`has_country_flag` 141, `is_greater_power` 130, `primary_culture` 128.

The cost is flat: the heaviest decision (`pay_owed_steamers`) is 0.29 ms/day, the top 25 together
2.5 ms. **[M]** Each decision is evaluated once per country per day.
