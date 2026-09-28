# actor-channels-kahn

> Move data between actors only over channels: `(chan n)` makes one, `chan-tx`/`chan-rx` are its two ends, `chan-send` blocks while it is full and `chan-recv` while it is empty. There is no select, no try-receive and no emptiness test, so the output is the same under every schedule.

## Why It Matters

Actors form a Kahn process network (`docs/concurrency-determinism-design.md`, spec §15, §27): each channel has exactly one writer and one reader, receive blocks, and a process cannot ask which channel is ready. A process's output is then a function of its inputs alone, whatever the scheduler does. The mailbox API (`send`, `receive`, `actor-self`, `actor-terminate`, `actor-send`) was removed on 2026-09-28; those names are `E_UNBOUND_VARIABLE` now.

| Form | Type |
|---|---|
| `(chan n)` | `Int -> (Chan a)`; n from 1 to 16777216, else `E_CHANNEL_CAPACITY` at run time |
| `(chan-tx c)`, `(chan-rx c)` | `(Chan a) -> (Tx a)`, `(Chan a) -> (Rx a)` |
| `(chan-send tx v)` | `(Tx a) a -> Unit`; blocks while the buffer is full |
| `(chan-recv rx)` | `(Rx a) -> a`; blocks while empty |

Channels are typed, so receiving an `Int` where a `String` is needed is `E_TYPE_MISMATCH` at compile time. For several kinds of message, send one ADT and `match` on it. Fan-in is one channel per producer, read in a fixed order.

- **Closing:** when the actor that owns a `Tx` finishes, the channel closes; `chan-recv` on a closed, drained channel is `E_CHANNEL_CLOSED` (catchable with `try`).
- **Deadlock:** when every live actor, `main` included, is blocked on a channel or a join, the program prints every actor's buffered output, then `PANIC: E_DEADLOCK: ...`, and exits 1. It is not catchable and is deterministic.

## Bad

```lisp
(send worker (Add 1))      ; E_UNBOUND_VARIABLE: send and receive are gone
(print (receive))
```

## Good

```lisp
(use actor/actor)

(defn main ()
  (let c (chan 2)
    (let tx (chan-tx c)
      (let rx (chan-rx c)
        (let a (spawn (fn () (begin (chan-send tx 20) (chan-send tx 22))))
          (let x (chan-recv rx)
            (let y (chan-recv rx)
              (begin
                (actor-wait a)
                (print (+ x y))     ; 42
                0))))))))
```

## Notes

- `chan`, `chan-send`, `chan-recv` and `spawn` need the `actor` capability in a package.
- Any value type may cross, closures included; a `let-mut` value may not ([actor-no-let-mut-crossing](actor-no-let-mut-crossing.md)).
- `ZYL_SCHED=deterministic` runs one actor at a time and `ZYL_SCHED_CHAOS=<seed>` perturbs every channel operation; the suite's `sched` category requires identical output under both. Use them to test actor code.
- The REPL and `zyl eval` run channels and actors too ([tool-repl](tool-repl.md)).
- Examples: `tests/regression/channels.zyl`, book chapters 9 and 21.

## See Also

- [actor-endpoint-ownership](actor-endpoint-ownership.md)
- [actor-always-wait](actor-always-wait.md)
- [actor-output-per-actor](actor-output-per-actor.md)
