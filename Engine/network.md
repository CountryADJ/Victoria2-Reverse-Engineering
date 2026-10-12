# Network layer (ENet) and the per-frame wait

Source: research of October 2026, found with a sampling profiler (suspend the game
thread every 1 ms, record its stack) and checked instruction by instruction in Ghidra. Marks: **[C]**
read in the code, **[M]** measured.

## Summary

- The game links **ENet** statically: `enet_host_service` `0x00A89D00`, `enet_socket_wait` `0x00A8B400`,
  identified by ENet's own error strings ("Error dispatching/sending/receiving packets") and by the
  `select` / `WSARecvFrom` imports at `0x00C8A330` / `0x00C8A338`. **[C]**
- Every frame `CommandQueue_Process` services the session object (`[obj+0x38]->vt[0x1C](0)` at
  `0x009AFBB1`), which ends in `NetSession_Service` (`0x00A61780`, virtual, slot at `0x00E42744`):
  `enet_host_service(this+0x18 host, &event, this+0x20 timeout)` at `0x00A618CC`. **[C]**
- The base session constructor (`0x00A5FF00`) sets the timeout to **15 ms**, the client/server variants
  (`0x00A635C0`, `0x00A63830`) to 35 ms. **[C]** A single-player game polled with 15 ms. **[M]**
  This happens **in single player too**, where the session is a local loopback.
- `enet_host_service` sends, receives and dispatches, then blocks in `select()` until a packet arrives or
  the timeout ends. **[C]**
- Measured (TGC 1836, max speed): the game thread spends **~46 ms per game day blocked in
  `ntdll!ZwWaitForSingleObject` under `select`**, about 23 % of a ~200 ms day. **[M]**

## Chain (sampled, checked instruction by instruction)

```text
CInGameIdler_Update
  CommandQueue_Process (0x009AF9F0)       CALL EAX at 0x009AFBB1   ([this+0x38]->vt[0x1C](0))
    ...                                    (frames in between not identified)
      NetSession_Service (0x00A61780)     CALL at 0x00A618CC
        enet_host_service (0x00A89D00)    CALL at 0x00A89E43
          enet_socket_wait (0x00A8B400)   CALL [select] at 0x00A8B477
            ws2_32 / mswsock -> ntdll!ZwWaitForSingleObject
```

This is also the "own time" earlier call-timing profiles attributed to `CommandQueue_Process`
(~51 ms/day): its virtual calls were not hooked, and almost all of it was this wait.
