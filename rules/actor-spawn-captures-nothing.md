# actor-spawn-captures-nothing

> Pass `spawn` a named zero-argument function or a zero-parameter `fn`; captures of immutable values work (e.g. an `(actor-self)` id to reply to), but a parameter is always 0 and a `let-mut` capture is `E_CAPABILITY_LEAK`.

## Why It Matters

`zyl_actor_spawn` unpacks a capturing closure into its code and environment, so a spawned `fn` that reads immutable variables from the enclosing scope works (captured by value, like any closure). The usual use is handing the worker a reply address: `(let me (actor-self) (spawn (fn () (worker me k))))`. A parameter on the entry receives 0; it is not a message (loop on `(receive)` instead). Capturing a `let-mut` is rejected at compile time (`E_CAPABILITY_LEAK`, located, with a label at the `let-mut`).

## Bad

```lisp
(spawn (fn (msg) (handle msg)))                ; msg is always 0
(let-mut c 0 (spawn (fn () (print c))))        ; E_CAPABILITY_LEAK
```

## Good

```lisp
(defn worker (reply-to k) (send reply-to (* k 2)))

(defn main ()
  (let me (actor-self)
    (let k 21
      (begin
        (spawn (fn () (worker me k)))   ; immutable captures are fine
        (print (receive))               ; 42
        0))))
```

## Notes

- Deliver data to a running actor with `send` + `(receive)` ([actor-send-is-discarded](actor-send-is-discarded.md)) or closure messages ([actor-closure-messages](actor-closure-messages.md)).
- Private actor state lives in `let-mut` locals (or loop parameters) of the entry function.

## See Also

- [actor-no-let-mut-crossing](actor-no-let-mut-crossing.md)
- [closure-capture-by-value](closure-capture-by-value.md)
