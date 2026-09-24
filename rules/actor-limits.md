# actor-limits

> Design within the runtime's limits: at most 1024 actors per process (ids never reused), ~8 MB actor stacks, unbounded mailboxes, and a panic in any actor kills the whole process.

## Why It Matters

| Limit | Consequence |
|---|---|
| `ZYL_MAX_ACTORS` = 1024 over the process lifetime | `spawn` prints `zyl: actor limit reached (1024)` and returns an invalid id |
| default pthread stack (~8 MB) | recursion that works in `main` (huge stack) can overflow in an actor |
| unbounded linked-list mailbox, mutex + condvar | no backpressure: build credit/ack into your protocol |
| panic in an actor | `exit(1)` for the whole process; no supervision or restart |
| REPL / `zyl eval` | cannot run actors (`E_UNSUPPORTED_INTERPRETED`) |

Costs (orders of magnitude): `spawn` ≈ 10 µs (one `pthread_create`), closure-message send ≈ 250 ns, `zyl_actor_wait_all` polls every 1 ms.

## Notes

- In a package, `spawn`/`send`/`receive` and any `stdlib/actor` use need the `actor` capability; the capability pass skips `main`'s body, but declare it anyway.
- Test pattern: keep logic in pure functions tested directly; test actors only for lifecycle.

## See Also

- [actor-closure-messages](actor-closure-messages.md)
- [pkg-capabilities](pkg-capabilities.md)
