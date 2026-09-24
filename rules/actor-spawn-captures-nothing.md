# actor-spawn-captures-nothing

> Pass `spawn` a named zero-argument function or a `fn` that captures nothing and takes no parameters.

## Why It Matters

The runtime's `zyl_actor_spawn(entry, state)` always receives state 0: the actor entry gets **no environment**. A spawned closure that reads any variable from the enclosing scope compiles and then **segfaults** (its environment block is used as the code pointer). A parameter on the entry receives 0; it is not a message. Capturing a `let-mut` is rejected at compile time (`E_CAPABILITY_LEAK`), but read-only captures are not.

## Bad

```lisp
(let x 41 (spawn (fn () (print (+ x 1)))))    ; compiles, crashes
(spawn (fn (msg) (handle msg)))                ; msg is always 0
```

## Good

```lisp
(use actor/actor)
(defn crunch (n) (if (<= n 1) 1 (* n (crunch (- n 1)))))
(defn report-a () (print (crunch 5)))

(defn main ()
  (let a (spawn (fn () (report-a)))   ; captures nothing
    (begin
      (actor-wait a)
      0)))
;; or (spawn report-a)
```

## Notes

- Deliver data to a running actor with closure messages ([actor-closure-messages](actor-closure-messages.md)).
- Private actor state lives in `let-mut` locals of the entry function.

## See Also

- [actor-no-let-mut-crossing](actor-no-let-mut-crossing.md)
- [closure-capture-by-value](closure-capture-by-value.md)
