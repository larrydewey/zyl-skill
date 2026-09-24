# actor-send-is-discarded

> Don't design around `send`/`receive`: data messages are queued and discarded, and there is no `receive`. Use closure messages to deliver work.

## Why It Matters

Actors are the least complete part of the language. `send` is asynchronous and FIFO and passes the message as a raw word (not copied), but the runtime **drops data messages** when it dequeues them; nothing in Zyl can observe them. `(receive)` is not implemented (a body using it does nothing). Sending to an actor that was already waited on or terminated is a no-op (it used to abort with `free(): invalid pointer`; fixed 2026-09-24).

## Bad

```lisp
(defn counter () (receive ...))     ; not implemented
(send worker (Inc 1))               ; queued, then discarded
(actor-wait a) (send a 1)           ; silently nothing
```

## Good

```lisp
(defn idle () 0)
(defn handler (msg) (print (* msg 10)))
(defn main ()
  (let a (spawn idle)
    (begin
      (ffi-call "zyl_actor_send_closure" a handler 1 1000)   ; runs handler(1) on a
      (ffi-call "zyl_actor_send_closure" a handler 2 1000)
      (ffi-call "zyl_actor_wait_all" 1000)                    ; drain, stop, join all
      0)))
;; 10, 20
```

## Notes

- `send` is still useful to exercise its compile-time checks.
- Closure messages need the `ffi` capability (in addition to `actor`) in a package.

## See Also

- [actor-closure-messages](actor-closure-messages.md)
- [actor-always-wait](actor-always-wait.md)
