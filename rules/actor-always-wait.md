# actor-always-wait

> Wait for every actor before `main` returns: `actor-wait` per actor, or end with `(ffi-call "zyl_actor_wait_all" 1000)`.

## Why It Matters

The generated `main` does **not** wait for actors. When `main` returns the process exits and running actors are killed (one test printed the actor's line in only 186 of 200 runs). An actor also does not stop when its entry function returns: it idles on its mailbox.

| Operation | Effect |
|---|---|
| `(actor-wait a)` | mark `a` stopped and join its thread; **queued closure messages are discarded** |
| `(actor-terminate a)` | mark `a` stopped (and join) |
| `(actor-is-alive a)` | true until waited/terminated |
| `(ffi-call "zyl_actor_wait_all" 1000)` | poll until **every** mailbox is empty, then stop and join every actor **in id order** |

## Bad

```lisp
(defn main ()
  (let a (spawn (fn () (print "hi from actor")))
    (print "main exits")))          ; actor output may never appear
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
        (print "all workers finished")   ; always last
        0))))
```

## Notes

- There is no `wait_all` language form (`(wait_all a)` is `E_UNBOUND_VARIABLE`).
- Use `zyl_actor_wait_all` when actors have pending closure messages; `actor-wait` drops them.

## See Also

- [actor-output-nondeterministic](actor-output-nondeterministic.md)
- [actor-closure-messages](actor-closure-messages.md)
