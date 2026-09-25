# actor-no-let-mut-crossing

> Never reference a `let-mut` variable in a `send` message or spawned closure; snapshot it with `let` first.

## Why It Matters

Values crossing an actor boundary must be Send-capable (spec R3). The check is syntactic (`mutability_check.zyl`): any `let-mut` name of the enclosing scope in a message or spawn body is `E_CAPABILITY_LEAK`, located at the `spawn`/`send` and naming the variable, with a label at its `let-mut`. A `Secret` crossing is `E_SECRET_ESCAPE`. The type checker has no Send predicate, so a closure, a `(Pin a)` or any other value the spec counts as non-Send passes if it is bound by plain `let`.

## Bad

```lisp
(let-mut x 10
  (send a x))
;; error[E_CAPABILITY_LEAK]: message sent to an actor references let-mut (TMut) variable `x` from the enclosing scope

(let-mut count 0
  (spawn (fn () (set! count (+ count 1)))))
;; error[E_CAPABILITY_LEAK]: spawned closure captures let-mut (TMut) variable `count` from the enclosing scope
```

## Good

```lisp
(let-mut x 10
  (let snapshot x
    (send a snapshot)))     ; sound for an Int: the word is copied
```

## See Also

- [actor-spawn-zero-arg-entry](actor-spawn-zero-arg-entry.md)
- [secret-five-prohibitions](secret-five-prohibitions.md)
