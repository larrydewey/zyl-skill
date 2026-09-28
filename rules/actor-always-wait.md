# actor-always-wait

> Join each actor with `actor-wait` where its output and its failure belong: the join emits the actor's buffered output and re-raises its panic. At exit, every unjoined actor is joined in spawn order and the first unjoined panic sets status 1.

## Why It Matters

An actor's uncaught panic ends only that actor; its message waits for the joiner. `(actor-wait a)` returns once `a` has finished, emits its output, and re-raises its panic, which `try` can catch. A second wait does nothing. When `main` returns, the runtime closes main's channels and joins every actor still unjoined, in spawn order; the first unjoined panic is printed after every actor's output and the exit status becomes 1. If `main` itself panics, the actors are abandoned.

| Operation | Effect |
|---|---|
| `(actor-wait a)` (`actor/actor`) | block until `a` finishes; emit its output; re-raise its panic |
| `(actor-is-alive a)` | a `Bool`: true until this program joins `a` (deterministic, never a race) |
| return from `main` | join every unjoined actor in spawn order |

## Bad

```lisp
(defn main ()
  (let _ (spawn (fn () (error "boom")))
    (begin (print "main done") 0)))
;; main done
;; PANIC: boom          <- reported at exit, status 1: nobody joined the actor
```

## Good

```lisp
(use actor/actor)
(defn main ()
  (let a (spawn (fn () (error "boom")))
    (begin
      (print (try (begin (actor-wait a) "ok") (catch e e)))   ; boom
      0)))
```

## Notes

- There is no `actor-terminate`: stopping an actor partway through would be a race.
- A join that can never finish (the actor waits on a channel only the joiner feeds) is `E_DEADLOCK`.

## See Also

- [actor-output-per-actor](actor-output-per-actor.md)
- [actor-channels-kahn](actor-channels-kahn.md)
