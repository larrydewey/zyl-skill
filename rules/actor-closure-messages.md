# actor-closure-messages

> Prefer `send` + `(receive)` for data messages; to run a function on a running actor, use `(ffi-call "zyl_actor_send_closure" actor handler word 1000)`, where `handler` is a named one-parameter function.

## Why It Matters

The primary message protocol is `send` + `(receive)` ([actor-send-is-discarded](actor-send-is-discarded.md)). Closure messages are the lower-level alternative: the actor needs no receive loop, and its thread runs queued calls `handler(word)` one at a time in queue order. `word` is an Int or a heap value passed **by pointer** (shared between threads, not copied — keep messages immutable). Request/response works by putting the requester's actor id in the message. In an actor that does call `receive`, closure messages queued ahead of the next data message run first.

## Good

```lisp
(deftype Msg (Request Int Int) (Reply Int))
(defn idle () 0)
(defn on-client (m)
  (match m
    ((Reply v) (print v))
    ((Request _ _) 0)))
(defn on-server (m)
  (match m
    ((Request from n)
      (ffi-call "zyl_actor_send_closure" from on-client (Reply (* n n)) 1000))
    ((Reply _) 0)))

(defn main ()
  (let server (spawn idle)          ; spawn the server FIRST (lower id)
    (let client (spawn idle)
      (begin
        (ffi-call "zyl_actor_send_closure" server on-server (Request client 7) 1000)
        (ffi-call "zyl_actor_wait_all" 1000)
        0))))
;; 49
```

## Reply-loss caveat

`zyl_actor_wait_all` now waits until every actor is parked on an empty mailbox before stopping any, instead of stopping them in id order. A reply can still be lost: running the example above 200 times printed `49` in 199 runs with the server spawned first and 194 with the client first (2026-09-24). Spawn repliers first, and treat a reply produced during the final drain as best-effort until the race is fixed.

## Notes

- A closure message to a stopped actor is dropped silently.
- `stdlib/actor/actor.zyl`: `actor-spawn`, `actor-send`, `actor-send-with-timeout` (timeout ignored), `actor-wait`, `actor-terminate`, `actor-is-alive`.
- Needs both `actor` and `ffi` capabilities in a package.

## See Also

- [actor-send-is-discarded](actor-send-is-discarded.md)
- [actor-always-wait](actor-always-wait.md)
