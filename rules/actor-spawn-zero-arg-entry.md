# actor-spawn-zero-arg-entry

> Pass `spawn` a named zero-argument function or a zero-parameter `fn`; immutable captures (channel ends above all) work, an entry with a parameter is `E_TYPE_MISMATCH`, and a `let-mut` capture is `E_CAPABILITY_LEAK`.

## Why It Matters

`spawn` has type `(() -> a) -> Actor`. The runtime unpacks a capturing closure into its code and environment, so a spawned `fn` reads immutable variables of the enclosing scope (captured by value, like any closure), and the channel ends it captures directly move to the new actor ([actor-endpoint-ownership](actor-endpoint-ownership.md)). The entry takes no parameters: give it its inputs by capture or over a channel. `spawn` returns an `Actor`, which is not an `Int`: `(+ a 1)` is `E_TYPE_MISMATCH`.

## Bad

```lisp
(spawn (fn (msg) (handle msg)))                ; E_TYPE_MISMATCH: the entry takes no parameters
(let-mut c 0 (spawn (fn () (print c))))        ; E_CAPABILITY_LEAK
```

## Good

```lisp
(use actor/actor)

(defn worker (rx tx) (chan-send tx (* 2 (chan-recv rx))))

(defn main ()
  (let jobs (chan 1)
    (let results (chan 1)
      (let r (chan-rx jobs)
        (let w (chan-tx results)
          (let a (spawn (fn () (worker r w)))   ; r and w move to the actor
            (begin
              (chan-send (chan-tx jobs) 21)
              (print (chan-recv (chan-rx results)))   ; 42
              (actor-wait a)
              0)))))))
```

## Notes

- Private actor state lives in `let-mut` locals (or loop parameters) of the entry function.

## See Also

- [actor-no-let-mut-crossing](actor-no-let-mut-crossing.md)
- [closure-capture-by-value](closure-capture-by-value.md)
