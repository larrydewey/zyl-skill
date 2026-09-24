# actor-always-wait

> Know how actors end: the process drains every actor at exit, but `actor-wait` drops an actor's queued closure messages. Wait explicitly only where you need an ordering point.

## Why It Matters

Every compiled program registers `zyl_actor_wait_all` as an `atexit` handler, so returning from `main` drains all actors' pending closure messages, then stops and joins them (the actor's line printed in 200 of 200 runs, 2026-09-24; before that fix, 186 of 200). An actor does not stop when its entry function returns: it idles on its mailbox until that drain or an explicit wait. An actor blocked in `(receive)` at exit also counts as idle, so the drain stops it instead of hanging.

| Operation | Effect |
|---|---|
| `(actor-wait a)` | mark `a` stopped and join its thread; **queued closure messages are discarded** |
| `(actor-terminate a)` | mark `a` stopped (and join) |
| `(actor-is-alive a)` | true until waited/terminated |
| `(ffi-call "zyl_actor_wait_all" 1000)` | poll until every mailbox is empty and every actor is parked, then stop and join all of them; also runs automatically at exit |

## Bad

```lisp
(use actor/actor)
(defn slow (m) (print m))
(defn idle () 0)
(defn main ()
  (let a (spawn idle)
    (begin
      (ffi-call "zyl_actor_send_closure" a slow 1 1000)
      (actor-wait a)                ; stops a now: 1 is never printed
      0)))
```

## Good

```lisp
(use actor/actor)
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
- Use `zyl_actor_wait_all` (or just return from `main`) when actors have pending closure messages; `actor-wait` drops them.
- The drain is not airtight: a reply sent while `zyl_actor_wait_all` is draining was lost in 1 to 6 of 200 runs (2026-09-24). See [actor-closure-messages](actor-closure-messages.md).

## See Also

- [actor-output-nondeterministic](actor-output-nondeterministic.md)
- [actor-closure-messages](actor-closure-messages.md)
