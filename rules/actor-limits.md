# actor-limits

> Design within the runtime's limits: at most 1024 actors per program (`E_ACTOR_LIMIT`), 8 MiB actor stacks, channel buffers of 1 to 16777216 values, one OS thread per actor.

## Why It Matters

| Limit | Consequence |
|---|---|
| 1024 actors over the program's life (ids never reused) | the 1025th `spawn` panics with `E_ACTOR_LIMIT` |
| 8 MiB stack per actor (guard page below) | recursion that works in `main` (a huge reserved stack) can overflow in an actor |
| channel capacity 1..16777216 | `(chan 0)` is `E_CHANNEL_CAPACITY` at run time; a bounded buffer is the backpressure |
| one thread per actor, one scheduler lock | fine for pipelines and worker pools; not for millions of tiny tasks |
| panic in an actor | ends only that actor; re-raised by `actor-wait` or reported at exit ([actor-always-wait](actor-always-wait.md)) |

Actors are the runtime's own `clone` threads in a freestanding program and pthreads in a hosted one. Allocation takes a lock only once a second thread exists, so the first `spawn` makes allocation slightly slower program-wide.

## Notes

- In a package, `spawn`, the channel forms and any `stdlib/actor` use need the `actor` capability (`E_PKG_CAPABILITY_VIOLATION`), `main` and top-level tests included.
- In the REPL the 1024 count covers the whole session.
- Test pattern: keep logic in pure functions tested directly; test actor wiring under the deterministic and chaos schedules.

## See Also

- [actor-channels-kahn](actor-channels-kahn.md)
- [pkg-capabilities](pkg-capabilities.md)
