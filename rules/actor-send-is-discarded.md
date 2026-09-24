# actor-send-is-discarded

> Exchange data messages with `send` + `(receive)`, and reply to `(actor-self)` ids (main included); a sent message is dropped only if the target never calls `receive`. Closure messages remain available.

## Why It Matters

`send` is asynchronous and FIFO per sender and passes the message as one 64-bit word: an Int or a pointer to an immutable heap value such as an ADT (not copied). `(receive)` returns the next **data** message in the running actor's mailbox, blocking until one arrives; closure messages queued ahead of it run first, so the mailbox stays FIFO. `(actor-self)` returns the running actor's id; on the main thread the first `actor-self` or `receive` opens a mailbox for `main` (no thread), so actors can reply to it. Structured messages are ADT values matched after `receive`.

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
- Closure messages (`zyl_actor_send_closure`, [actor-closure-messages](actor-closure-messages.md)) still work and interleave FIFO with data messages.
- The REPL / `zyl eval` interpreter does not support actors.
- Examples: `book/examples/actor-counter/counter.zyl`, `tests/regression/actor-receive.zyl`; book §21.3 "Receiving".

## See Also

- [actor-closure-messages](actor-closure-messages.md)
- [actor-always-wait](actor-always-wait.md)
- [actor-spawn-captures-nothing](actor-spawn-captures-nothing.md)
