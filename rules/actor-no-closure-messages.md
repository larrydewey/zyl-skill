# actor-no-closure-messages

> Use `send` + `(receive)` for every message: the runtime's closure messages (`zyl_actor_send_closure`) cannot be sent from a Zyl program.

## Why It Matters

The runtime has two message kinds. Data messages come from `send` and are returned by `(receive)` ([actor-send-is-discarded](actor-send-is-discarded.md)). Closure messages, queued by the C entry `zyl_actor_send_closure(actor, fn, state)`, make the actor's thread call `fn(state)` when it reaches them. Nothing in the compiler or the standard library sends one, and a program cannot: the entry is a runtime export with no signature in `stdlib/compiler/ffi_sigs.zyl`, so the call is `E_CANNOT_INFER` ("no type for untyped ffi result"), and declaring it with `extern` is `E_FFI_RESTRICTED` (an extern may not retype a runtime entry). Code written for the older, untyped compiler that passed a handler function this way no longer compiles.

## Bad

```lisp
(defn slow (m) (print m))
(defn idle () 0)
(defn main ()
  (let a (spawn idle)
    (begin
      (ffi-call "zyl_actor_send_closure" a slow 1 1000)   ; E_CANNOT_INFER
      0)))

(extern "zyl_actor_send_closure" (Actor (Fn (Int) Unit) Int) Unit)   ; E_FFI_RESTRICTED at each call
```

## Good

Request and reply as data messages, with the requester's `Actor` id in the message:

```lisp
(deftype Msg (Request Actor Int) (Reply Int))

(defn server ()
  (match (receive)
    (Request from n (send from (Reply (* n n))))
    (Reply _ unit)))

(defn main ()
  (let me (actor-self)
    (let s (spawn server)
      (begin
        (send s (Request me 7))
        (match (receive)
          (Reply v (print v))                  ; 49
          (Request _ _ unit))
        0))))
```

## Notes

- `(receive)` still runs any closure message queued ahead of the next data message, and an actor whose entry has returned runs them from its idle loop; this matters only to C code that calls the runtime directly.
- `stdlib/actor/actor.zyl`: `actor-spawn`, `actor-send`, `actor-send-with-timeout` (returns `Ok`; the timeout is ignored, since send never blocks), `actor-wait`, `actor-terminate`, `actor-is-alive`.

## See Also

- [actor-send-is-discarded](actor-send-is-discarded.md)
- [actor-always-wait](actor-always-wait.md)
- [ffi-extern-required](ffi-extern-required.md)
