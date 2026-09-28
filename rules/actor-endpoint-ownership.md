# actor-endpoint-ownership

> Each channel end belongs to one actor at a time. The creator owns both; an end moves only when a spawned closure captures it directly, or when it is itself sent over a channel. Using an end you do not own is `E_CHANNEL_NOT_OWNER`.

## Why It Matters

Single writer, single reader is what makes the network deterministic, so the runtime enforces it. Ownership changes only at those two program points, so the error is deterministic too. After `spawn` captures `tx`, the spawner can no longer send on it.

An end nested inside a captured value (a struct or variant field) does **not** move at spawn: the actor's use of it is `E_CHANNEL_NOT_OWNER`, and the program often then stops with `E_DEADLOCK`. Capture the end itself, or send it over a channel.

## Bad

```lisp
(let tx (chan-tx c)
  (let a (spawn (fn () (chan-send tx 1)))   ; tx moves to the actor here
    (chan-send tx 2)))                        ; E_CHANNEL_NOT_OWNER

(deftype Box (Box (Tx Int)))
(let b (Box (chan-tx c))
  (spawn (fn () (match b (Box t (chan-send t 1))))))   ; t never moved: E_CHANNEL_NOT_OWNER
```

## Good

```lisp
;; hand a reply channel's Tx to a worker over another channel
(let c (chan 1)
  (let d (chan 1)
    (let dtx (chan-tx d)
      (let crx (chan-rx c)
        (let a (spawn (fn () (let t (chan-recv crx) (chan-send t 7))))
          (begin
            (chan-send (chan-tx c) dtx)          ; dtx now belongs to whoever receives it
            (print (chan-recv (chan-rx d)))      ; 7
            (actor-wait a)
            0))))))
```

## Notes

- An end in transit (sent, not yet received) has no owner.
- The interpreter moves the ends an interpreted closure captures the same way (`zyl_chan_spawn_moves`).

## See Also

- [actor-channels-kahn](actor-channels-kahn.md)
- [actor-spawn-zero-arg-entry](actor-spawn-zero-arg-entry.md)
