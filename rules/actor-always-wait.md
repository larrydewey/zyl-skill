# actor-always-wait

> Know how actors end: returning from `main` drains every actor, while `actor-wait` stops one at once and drops the messages still queued for it. Wait explicitly only where you need an ordering point.

## Why It Matters

Every compiled program registers `zyl_actor_wait_all` as an `atexit` handler, so returning from `main` waits until every mailbox is empty and every actor is parked, then stops and joins them all (the actor's line printed in 200 of 200 runs, 2026-09-24; before that fix, 186 of 200). An actor does not stop when its entry function returns: it idles on its mailbox until that drain or an explicit wait. An actor blocked in `(receive)` at exit also counts as idle, so the drain stops it instead of hanging.

| Operation | Effect |
|---|---|
| `(actor-wait a)` (`actor/actor`) | mark `a` stopped and join its thread; **messages still queued are discarded** |
| `(actor-terminate a)` | mark `a` stopped (and join) |
| `(actor-is-alive a)` | a `Bool`: true until waited/terminated |
| `(ffi-call "zyl_actor_wait_all" 1000)` | poll until every mailbox is empty and every actor is parked, then stop and join all of them; also runs automatically at exit. A runtime entry typed `-> Unit`, so it needs no extern |

## Bad

```lisp
(use actor/actor)
(defn printer () (print (receive)))
(defn main ()
  (let a (spawn printer)
    (begin
      (send a 1)
      (actor-wait a)                ; stops a now: 1 was never printed in 50 runs
      0)))
```

## Good

```lisp
(use actor/actor)
(defn job-a () (print "a"))
(defn job-b () (print "b"))
(defn main ()
  (let a (spawn (fn () (job-a)))
    (let b (spawn (fn () (job-b)))
      (begin
        (actor-wait a)
        (actor-wait b)
        (print "all workers finished")   ; after both, because of the explicit waits
        0))))
```

## Notes

- There is no `wait_all` language form (`(wait_all a)` is `E_UNBOUND_VARIABLE`).
- Use `zyl_actor_wait_all` (or just return from `main`) when actors still have messages to handle; `actor-wait` drops them.
- To know an actor has finished its work, have it `send` a reply to `(actor-self)` of the waiter and `receive` it; that is an ordering point that loses nothing.
- The drain is not airtight: a reply sent while `zyl_actor_wait_all` is draining was lost in 1 to 6 of 200 runs (2026-09-24). See [actor-no-closure-messages](actor-no-closure-messages.md).

## See Also

- [actor-output-nondeterministic](actor-output-nondeterministic.md)
- [actor-no-closure-messages](actor-no-closure-messages.md)
