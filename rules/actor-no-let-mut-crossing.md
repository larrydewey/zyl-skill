# actor-no-let-mut-crossing

> Never reference a `let-mut` variable in a `chan-send` value or a spawned closure; snapshot it with `let` first.

## Why It Matters

Values crossing an actor boundary must be Send-capable (spec R3). The check is syntactic (`mutability_check.zyl`): any `let-mut` name of the enclosing scope in a sent value or spawn body is `E_CAPABILITY_LEAK`, located at the `spawn`/`chan-send` and naming the variable, with a label at its `let-mut`. A `Secret` crossing is `E_SECRET_ESCAPE`. The type checker has no Send predicate, so a closure, a `(Pin a)` or any other value the spec counts as non-Send passes if it is bound by plain `let`.

## Bad

```lisp
(capabilities actor)
(defn main ()
  (let tx (chan-tx (chan 1))
    (let-mut x 10
      (begin (chan-send tx x) 0))))   ; E_CAPABILITY_LEAK: the sent value names let-mut `x`
```

```lisp
(capabilities actor)
(defn main ()
  (let-mut count 0
    (let _ (spawn (fn () (set! count (+ count 1)))) 0)))   ; E_CAPABILITY_LEAK: the closure captures `count`
```

## Good

```lisp
(capabilities actor)
(use actor/actor)
(defn main ()
  (let c (chan 1)
    (let-mut x 10
      (let snapshot x
        (begin
          (chan-send (chan-tx c) snapshot)     ; sound for an Int: the word is copied
          (print (chan-recv (chan-rx c)))      ; 10
          0)))))
```

## See Also

- [actor-spawn-zero-arg-entry](actor-spawn-zero-arg-entry.md)
- [secret-five-prohibitions](secret-five-prohibitions.md)
