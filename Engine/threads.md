# Threads

What already runs in parallel in `v2game.exe`, and why the simulation does not.

Source: research of October 2026, building on the worker framework and accumulator
notes in [../Economy/economy-market.md](../Economy/economy-market.md). Marks: **[C]** read in the code,
**[I]** inferred, **[M]** measured.

## Summary

- The engine links Intel TBB (`tbb.dll`). `task_scheduler_init::initialize(threads)` is called with -1
  (TBB default); the thread count only matters for the few `parallel_for` sites. **[C]**
- Known `parallel_for` sites **[C]**:
  - `FUN_0068F4C0`, called once a day from `DayTick` (`0x00685FD6`): `start_for<blocked_range<unsigned>,
    ProcessPopStatisticsFunctor, auto_partitioner>`. The functor runs `Economy_CalculateDistribution`
    (`0x00520150`) -> `Economy_AccumulateElement` (`0x00969760`) over the pops. About 4 ms/day of it lands
    on the game thread in 1836 with 4 TBB threads. **[M]**
  - entity/spatial registration around `Entity_RegisterSpatial` (`0x0068F390`, 19 047 calls/day);
  - graphics near `Render_InitRestore` (`0x0099DA20`).
- The rest of the daily simulation is single-threaded.
- A small generic worker-thread class exists (vtable `0x00E439F8`): `WorkerThread_Create` (`0x00A7AED0`)
  calls `CreateThread` with `WorkerThread_Entry` (`0x00A7B0C0`), which dispatches to `[this+0x18]->vt[1]`
  (`WorkerThread_Dispatch` `0x00A7B090`). Which job objects use it is unknown.

## Why the big daily loops are not parallel-safe **[C]**

Checked block by block:

| block | blocker |
|---|---|
| `Economy_CalculateDistribution` / `Economy_AccumulateElement` | write shared 64-bit accumulators (`this+0x8D8+i*8`, `this+0x900+i*8`, `this+0x13E8/+0x13EC`, `+=` on one object); the game's own `parallel_for` must already accept that or partition it |
| `WorldMarketUpdate` (`0x00484060`) / `Market_UpdateCountryPops` (`0x00485E40`) | a Mersenne Twister draw in the per-state work (`FUN_004A46E0` -> `FUN_004A5170` -> `FUN_009B7950` -> `Rng_Next`); static scratch buffers rewritten per country and per pop (`0x013F24F4`…`0x013F25A8`); `UpdateTimeThrottle` inside the loop can render a frame; sums clamp at 99 999 (not associative) |
| `Country_Recalculate` (`0x00515490`) | creates and registers objects (`FUN_005163C0` + vt `+0x28` + `FUN_004A7BC0`), unlinks and deletes objects from world lists (`world+0xA84`, `FUN_005F2AE0`), writes shared relation objects (`+0x38`, `+0x71`) |
| `Country_PrePass` (`0x00508590`) | writes other countries' relations (`+0x40/+0x44`) and treasuries (`other+0xE78`) |

Trigger evaluation, on the other hand, behaves as read-only: see
[triggers-and-scopes.md](triggers-and-scopes.md#thread-safety-of-trigger-evaluation).

## Floating point state

Threads created outside the game do not inherit its x87 control word or MXCSR. Direct3D 9 may lower the
x87 precision on the thread that created the device, so code that evaluates game floats on another
thread must copy both from the game thread first. **[I]**
