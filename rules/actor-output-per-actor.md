# actor-output-per-actor

> An actor's `print`s go to its own buffer, emitted when it is joined (`actor-wait`) or at exit in spawn order; main's output goes straight to stdout. So actor output is deterministic, and it appears where the join is.

## Why It Matters

Output is part of the determinism contract (spec §27): the same program prints the same bytes under every schedule. Each owner has a stdout buffer and a stderr buffer (stderr emitted after stdout); `file-write` to fd 1 or 2 from an actor goes through them too. Only a foreign C `write` bypasses them. A consequence: an actor's lines appear at the join, after anything main printed before it, however early the actor printed them.

## Good

```lisp
(use actor/actor)
(defn main ()
  (let a (spawn (fn () (print "actor")))
    (begin
      (print "main first")
      (actor-wait a)       ; "actor" is emitted here
      (print "main last")
      0)))
;; main first
;; actor
;; main last
```

## Notes

- A nested join lands in the joiner's buffer, so output nests the way joins do.
- Test actor code under `ZYL_SCHED=deterministic` and a few `ZYL_SCHED_CHAOS=<seed>` values and compare the output byte for byte.

## See Also

- [actor-always-wait](actor-always-wait.md)
- [det-nondeterminism-sources](det-nondeterminism-sources.md)
