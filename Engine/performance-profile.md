# Performance profile: where a game day goes

Measured cost of the daily tick and the frame at max speed, the instructions the game waits on, and the
data layouts behind them.

Source: research of October 2026. Two tools:
- a **call-timing profiler** (redirects `CALL rel32` sites and vtable slots, times them; levels 1–5);
- a **sampling profiler** (every 1 ms: suspend the game thread, read `EIP`, walk the stack, resume),
  which also sees virtual calls, other DLLs and the exact instruction being waited on.

TGC mod, new game in 1836 unless stated, max speed. Marks: **[M]** measured, **[C]** read in the code.

## Budget (vanilla, 1836, ~196 ms/day) **[M]**

| block | ms/day | share |
|---|---|---|
| frame side, outside `Update_World_and_Advance_Time` | ~90 | ~45 % |
| ↳ `CommandQueue_Process` own time: the ENet poll blocking in `select` | ~46–51 | 25 % |
| ↳ `AI_FrameUpdate` (the AI queue, mostly the decision loop) | ~24–26 | 12 % |
| ↳ `Render_MapFrame` (`0x006592F0`) | ~14.5 | 7 % |
| `WorldMarketUpdate` (inclusive) | ~23 | 11 % |
| `Country_PrePass` (`0x00508590`) | ~27 | 13 % |
| `Country_DailyUpdate` own | ~20 | 10 % |
| `ProcessMarketAndGoodsDistribution` | ~8.7 | 4 % |
| AI casus belli scans | ~11 | 5 % |
| events (`EventManager_DailyScan`) | ~4 | 2 % |
| provinces, `DayTick` own, rest | ~10 | 5 % |

A day takes 3–4 frames at max speed ([daily-tick.md](daily-tick.md#frames-per-day-the-ai-job-queue)).
About one frame a day is rendered from inside the tick by `UpdateTimeThrottle` (~5 ms/day). The ENet
wait is explained in [network.md](network.md).

## Where the game waits: one instruction per function **[M]**

Share of each function's samples on its single hottest instruction. A high share right after a load
means the CPU is stalled on that load (a cache miss); the sample lands on the instruction after it.

| function | ms/day own | top instruction | what stalls |
|---|---|---|---|
| `FUN_00522CF0` (market capacity) | 5.6 | 90 % `0x00522E27` | relation `+0x34` |
| `AI_TargetScan` `0x0084B0F0` | 3.9 | 90 % `0x0084B29D` | relation `+0x34` |
| `CountryRelation_DailyPurge` `0x0092E290` | 5.6 | 71 % | relation `+0x50` (73 %), `+0xA8` (18 %) |
| `Country_PrePass` | 3.6 | 70 % `0x00508DEE` | relation `+0x2C` |
| `Country_Recalculate` `0x00515490` | 7.8 | 46 % `0x00515FCF` | relation `+0x20` (46 %), `+0x70` (14 %) |
| `FUN_004B9C60` (DayTick pop pass) | 1.3 | 89 % `0x004B9C9B` | pop `+0x27C` (next) |
| `FUN_00485390`, `Market_UpdateCountryPops` | 4.5 | 32–34 % | first read of each pop |
| `FUN_0095EB00` (market distribution) | 1.4 | 90 % `0x0095EB63` | pop `+0x11C` |
| `FUN_0095A5F0` | 2.1 | 46 % `0x0095A66D` | pop `+0x180..+0x21C` writes |
| `FUN_0052FCD0` | 1.9 | 48 % `0x00530388` | `other+0x1580[self]` |
| `FUN_009E6C10` (`has_country_flag` trie) | 2.2 | 67 % `0x009E6C2F` | one trie node per character |
| `Economy_AccumulateElement` `0x00969760` | 2.4 | 55 % `0x00969889` | pop `+0xA4` vector |

(1836, sampling profiler.)

## The data layouts behind the stalls **[C]**

**Relations.** `country+0xBE8` is an array of 683 pointers, one per tag (indexed by the other country's
`+0x20`); each points to a separately allocated relation object, and every pair exists (the code reads
them without null checks), so there are ~683 × 683 ≈ 466 000 of them. At least four daily loops walk
"this country × every other country" and read one or two fields of each:

| loop | fields | visits |
|---|---|---|
| `Country_Recalculate` | `+0x08 +0x0C +0x20 +0x24 +0x28 +0x34 +0x38 +0x70` | countries with provinces |
| `FUN_00522CF0` (from `Market_AddRealDemand`, `0x00487483`) | `+0x34` | `world+0xADC` countries with provinces |
| `CountryRelation_DailyPurge` (from `Country_DailyUpdate`, `0x0050AC44`) | `+0x50 +0xA8 +0x2C +0x30 +0x28 +0x20` | countries with provinces |
| `Country_PrePass` loop (`0x00508DA7`) | `+0x2C`, `+0x30` | every tag with an index (dead ones too) |
| `AI_TargetScan` | `+0x34`, `+0x3C` | every tag 1..count−1 (dead ones too) |

The `Country_PrePass` and `AI_TargetScan` bodies depend on the relation data and write other countries
(PrePass moves money into `other+0xE78` when relation `+0x2C`/`+0x30` are set), so tags without
provinces still matter whenever their relation fields are set.

**Country pair array.** `country+0x1580` is an array of 8-byte values indexed by country index;
`FUN_0052FCD0` reads `other+0x1580[self]` for every living country.

**Living countries.** `world+0xADC..+0xAE0` is a `vector<CCountry*>` walked by `FUN_00522CF0` and
`FUN_0052FCD0`.

**Pops.** `province+0x194` holds 16 bytes per pop type; `+0` is the head of a linked list of pops of that
type, next at `pop+0x27C` (pop objects are at least 0x280 bytes). See
[../Economy/economy-daily-path.md](../Economy/economy-daily-path.md).

## Memory allocator

The VS2010 CRT is statically linked: `_malloc` `0x00AB049C`, `_free` `0x00AAEED7`, `_realloc` `0x00AC4320`,
`__calloc_impl` `0x00ABF700`, `__msize` `0x00AB71DB`, `operator new` `0x00AAE9AF`, `operator delete`
`0x00AAE91B`, one private heap created in `__heap_init` (`0x00ABABD8`). **[C]** Time in ntdll called
directly from `_malloc`/`_free` is ~1.7 ms/day in mid game; Windows enables the low-fragmentation heap on
it, so same-size objects (relations, pops) are already grouped by size class. **[M]** The cache misses
above come from the access pattern, not from the allocator.

## Pitfalls of sampling this game

- **ASLR.** The executable has a relocation table and `DYNAMIC_BASE`; it was seen at `0x00080000`.
  Convert runtime addresses with `va = runtime − base + 0x00400000` before matching them to a listing.
- **Stale return addresses.** Uninitialised locals keep old return addresses; after indirect calls
  they pass a "follows a `CALL`" check and build false chains (e.g. `UI_GetState` appearing to call
  `AI_FrameUpdate`). Walking the `EBP` chain first and validating `CALL rel32` targets removes most.
- **Bounds checks in the stack walk** must not overflow: frame values near `0xFFFFFFFF` occur in late
  game.
