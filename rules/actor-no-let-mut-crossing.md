# actor-no-let-mut-crossing

> Never reference a `let-mut` variable in a `send` message or spawned closure; snapshot it with `let` first.

## Why It Matters

Values crossing an actor boundary must be Send-capable (`TCap`/`TAtomic`). The check is syntactic (`mutability_check.zyl`): any `let-mut` name of the enclosing scope in a message or spawn body is `E_CAPABILITY_LEAK` (unlocated `PANIC:` line). A `Secret` crossing is `E_SECRET_ESCAPE`. The type-level Send predicate is not called, so `TBox`/`TPin`/non-Send fields are not rejected.

## Bad

```lisp
(let-mut x 10
  (send a x))
;; PANIC: E_CAPABILITY_LEAK: message sent to an actor references a let-mut (TMut) variable ...

(let-mut count 0
  (spawn (fn () (set! count (+ count 1)))))
;; PANIC: E_CAPABILITY_LEAK: spawned closure captures a let-mut (TMut) variable ...
```

## Good

```lisp
(let-mut x 10
  (let snapshot x
    (send a snapshot)))     ; sound for an Int: the word is copied
```

## See Also

- [actor-spawn-captures-nothing](actor-spawn-captures-nothing.md)
- [secret-five-prohibitions](secret-five-prohibitions.md)
