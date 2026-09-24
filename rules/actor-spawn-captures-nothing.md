# actor-spawn-captures-nothing

> Pass `spawn` a named zero-argument function or a zero-parameter `fn`; read-only captures work, parameters and `let-mut` captures do not.

## Why It Matters

`zyl_actor_spawn` unpacks a capturing closure into its code and environment, so a spawned `fn` that reads variables from the enclosing scope works (captured by value, like any closure; verified 2026-09-24, before which it segfaulted). A parameter on the entry receives 0; it is not a message. Capturing a `let-mut` is rejected at compile time (`E_CAPABILITY_LEAK`, located, with a label at the `let-mut`).

## Bad

```lisp
(spawn (fn (msg) (handle msg)))                ; msg is always 0
(let-mut c 0 (spawn (fn () (print c))))        ; E_CAPABILITY_LEAK
```

## Good

```lisp
(use actor/actor)
(defn crunch (n) (if (<= n 1) 1 (* n (crunch (- n 1)))))
(defn report-a () (print (crunch 5)))

(defn main ()
  (let n 5
    (let a (spawn (fn () (print (crunch n))))   ; read-only capture is fine
      (begin
        (actor-wait a)
        0))))
;; or (spawn report-a)
```

## Notes

- Deliver data to a running actor with closure messages ([actor-closure-messages](actor-closure-messages.md)).
- Private actor state lives in `let-mut` locals of the entry function.

## See Also

- [actor-no-let-mut-crossing](actor-no-let-mut-crossing.md)
- [closure-capture-by-value](closure-capture-by-value.md)
