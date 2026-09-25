# actor-send-is-discarded

> Exchange data messages with `send` + `(receive)`, and reply to `(actor-self)` ids (main included); a sent message is dropped only if the target never calls `receive`, and `receive`'s result is not type-checked.

## Why It Matters

`send` takes an `Actor` and a message, returns Unit, is asynchronous and FIFO per sender, and passes the message as one 64-bit word: an Int or a pointer to an immutable heap value such as an ADT (not copied). `(receive)` returns the next **data** message in the running actor's mailbox, blocking until one arrives; closure messages queued ahead of it run first, so the mailbox stays FIFO. `(actor-self)` returns the running actor's id; on the main thread the first `actor-self` or `receive` opens a mailbox for `main` (no thread), so actors can reply to it. `(actor-self)` and `spawn` have type `Actor`, so a message field carrying a reply address is typed `Actor` (`(Get Int)` is rejected at `(send reply-to ...)`). Structured messages are ADT values matched after `receive`.

**`receive` is not type-checked**: it has type `-> a`, whatever its use needs (the one known hole in the sound checker; typed channels are planned to close it). Nothing checks that senders sent that type, so a mismatch is a wrong value at run time. Give each actor one message type and keep its senders to it.

Data messages sent to an actor that never calls `receive` are still dropped when its loop drains them. Sending to an id that is not a live actor does nothing. An actor blocked in `receive` at program exit counts as idle and the exit drain stops it (no hang), but `(receive)` on `main` with nobody sending blocks forever.

## Bad

```lisp
(defn idle () 0)
(let a (spawn idle) (send a (Inc 1)))   ; idle never receives: dropped
(print (receive))                        ; on main, nobody sends: hangs forever
```

## Good

```lisp
(deftype CounterMsg (Add Int) (Get Int) (Stop Int))

(defn counter-loop (total)
  (match (receive)
    (Add n (counter-loop (+ total n)))
    (Get reply-to (begin (send reply-to total) (counter-loop total)))
    (Stop reply-to (send reply-to total))))

(defn main ()
  (let me (actor-self)
    (let c (spawn (fn () (counter-loop 0)))
      (begin
        (send c (Add 5))
        (send c (Add 7))
        (send c (Get me))
        (print (receive))          ; 12
        (send c (Stop me))
        (print (receive))          ; 12
        0))))
```

## Notes

- `receive`, `send` and `actor-self` need the `actor` capability in a package.
- Closure messages (`zyl_actor_send_closure`) cannot be sent from Zyl code; see [actor-no-closure-messages](actor-no-closure-messages.md).
- The REPL / `zyl eval` interpreter does not support actors.
- Examples: `book/examples/actor-counter/counter.zyl`, `tests/regression/actor-receive.zyl`; book §21.3 "Receiving".

## See Also

- [actor-no-closure-messages](actor-no-closure-messages.md)
- [actor-always-wait](actor-always-wait.md)
- [actor-spawn-zero-arg-entry](actor-spawn-zero-arg-entry.md)
