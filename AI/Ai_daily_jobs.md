# AI daily jobs (the AI queue)

How the per-country AI work is queued at the end of each day and drained over the following frames,
and what one country's job does.

Source: research of October 2026, static reading plus a sampling profiler on TGC 1836
and 1848–1855. Marks: **[C]** read in the code, **[M]** measured.

## The queue **[C]**

- At the end of `DayTick` (`0x006859C0`), for every AI object in `world+0xA4` whose country has
  provinces, a node `{tag, country index, prev, next, byte flag}` is appended to bucket
  `country % N` of the queue at `world+0xB84` (buckets are 16 bytes `{head, tail, count, flag}`; N = 1 in
  single player, the bucket count is `(world+0xB88 − world+0xB84) / 16`).
- `CInGameIdler_Update` calls `AI_FrameUpdate` (`0x006874C0`, stdcall, argument = pointer to the bucket
  array) at `0x006557C4` when bucket 0 is not empty.
- `AI_FrameUpdate` pops nodes; for a country that still has provinces and an AI object
  (`country+0x208 != 0`) it runs `AI_CountryFrameJob` (`0x0081E9E0`). It stops when the bucket is empty
  **or when 0.02 s have passed** (double at `0x00E456C8`, compared with a timer at `DAT_01258A8C` vt `+0x14`).
- `ProcessGameTimeAdvance` (`0x00682BD0`) issues the next day only when every bucket is empty.

At max speed this makes a day span several frames (3–4 in 1836), each with its own render and UI pass.
**[M]** The 20 ms budget is wall time, so in vanilla the frame boundaries, and which queued commands run
between AI jobs, depend on machine speed.

## One country's job: `AI_CountryFrameJob` (0x0081E9E0) **[C][M]**

| part | ms/day (1836) | |
|---|---|---|
| `AI_DecisionUpdate` (`0x00821190`) | ~10.7 | the daily decision loop, see [../Engine/decisions.md](../Engine/decisions.md) |
| `AI_CountryDailyJob_Update` (`0x0084B690`, vtable slot `0x00E27738`) | ~9.7 | below |

`AI_CountryDailyJob_Update` (`__fastcall`, AI object; `+0x38` country index):

- when `(FUN_00424930() + country index) % 15 == 0` (most likely the day, i.e. every 15 days, staggered by country) or when `+0x1BD` is set: `FUN_00851440`;
- sums a float over the child objects of `+0x13C` (vt `+0x15C`) against the country's money
  (`country+0xE78`, 2^15 fixed point) and adjusts `+0x1C8`;
- updates the sub-AIs in `+0x44` and `+0x214` (vt `+0x24`), calls `FUN_00827F30`, runs `FUN_0084EF30` eight
  times;
- reaches `AI_TargetScan` (`0x0084B0F0`, vtable slot `0x00E2773C`) and `FUN_00826480` (~4.1 ms/day, no
  single hot spot).

### `AI_TargetScan` (0x0084B0F0)

For every country (index 1..count−1, **with or without provinces**) it tests
`CMP [self+0xBE8[other] + 0x34], 0` (the same relation field market capacity `FUN_00522CF0` reads), plus
tag checks against `REB`, a search of `self+0x4A4` and, for neighbours on the same continent
(`capital province +0x16C`), a test of relation `+0x3C < -50000` before adding the country to target
lists (`FUN_004D61D0`). **90 % of its samples sit right after the `+0x34` read** (`0x0084B29D`): one
relation cache miss per country per AI country per day. 3.9 ms/day of own time in 1836. **[M]**

Its body depends on relation data and adds entries to lists for tags without provinces too:
a dead tag with a non-zero `+0x34` would still be added.
