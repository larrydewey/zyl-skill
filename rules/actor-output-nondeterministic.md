# actor-output-nondeterministic

> Let exactly one actor (usually `main`) produce ordered output, or collect results and print after `zyl_actor_wait_all`.

## Why It Matters

The spec promises deterministic actor output; the implementation does not deliver it. Every actor is its own pthread scheduled by the OS. Two actors printing 200 lines each produced 16 different outputs in 20 runs; three fan-out workers printed out of order in 1 of 100 runs. What does hold: FIFO from one sender to one actor, and one message at a time per actor. To order results, have workers `send` them to `main` (an `(actor-self)` id) and print in the order `main` `(receive)`s them.

## Good

```lisp
;; serialize when order matters
(let a (spawn (fn () (report-a)))
  (begin
    (actor-wait a)
    (let b (spawn (fn () (report-b)))
      (begin (actor-wait b) 0))))
```

## Notes

- Tests involving actors should assert on lifecycle (`actor-is-alive` after `actor-wait`), not on interleaved output.
- There is no deterministic scheduler, quantum, pool or stack-size knob. A Kahn-network redesign (single-sender channels, blocking receive, a `--sched=deterministic` oracle) is planned in PROGRESS.md but not implemented.

## See Also

- [det-nondeterminism-sources](det-nondeterminism-sources.md)
- [actor-always-wait](actor-always-wait.md)
