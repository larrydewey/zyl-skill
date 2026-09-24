# data-collections-persistent

> `Vec` is generic (`(Vec T)`), `core/map` is `(Map String V)`, and `collections/map`/`collections/set` hold `Int`s; every operation returns an updated value: always rebind to the result and treat the old value as used up.

## Why It Matters

`collections/vec` is an arena-backed generic ADT, `(deftype Vec (VecC Int Int Int Int T))` with a phantom `T`: elements are one word each, so any value fits, and `vec-get` returns `T`, so a String element prints and compares as text. Operations return a new `Vec` that **shares the buffer** with the old one: `vec-push` onto an old version is visible through newer ones. `collections/map` and `collections/set` are Int-keyed, Int-valued arena tables with the same sharing (`map-put` on an existing key and `map-remove`/`set-remove` compact in place). Out-of-range reads return sentinels, not errors.

## Bad

```lisp
(use collections/vec)
(let v (vec-create 0 10)
  (begin
    (vec-push v "a")          ; result discarded: v still has length 0
    (print (vec-len v))))     ; 0

(let v1 (vec-push (vec-create 0 4) 1)
  (let v2 (vec-push v1 2)     ; v1 and v2 share storage
    (vec-push v1 99)))        ; overwrites v2's element 1
```

## Good

```lisp
(use collections/vec)
(use core/map)
(defn main ()
  (let-mut v (vec-create 0 10)
    (begin
      (set! v (vec-push v "hello"))
      (set! v (vec-push v "world"))
      (print (vec-get v 1))                          ; world
      (print v)                                      ; [hello, world]
      (print (map-insert (map-new) "k" (vec-get v 0)))  ; {k: hello}
      0)))
```

## API cheatsheet

| Vec (`collections/vec`) | Map (`core/map`) | Int map (`collections/map`) | Int set (`collections/set`) |
|---|---|---|---|
| `(vec-create arena cap)` (arena 0 = private) | `(map-new)` | `(map-create arena cap)` | `(set-create arena cap)` |
| `vec-create-default cap` | | `map-create-default cap` | |
| `vec-push v x` (grows) | `map-insert m k v` (replaces) | `map-put m k v` | `set-add s k` |
| `vec-get v i` (OOB: the word `-1`) | `map-get m k` → `Option`; `map-get-or` | `map-get m k default` | `set-contains s k` |
| `vec-set v i x` (past len within cap extends) | `map-has`, `map-remove` | `map-has`, `map-remove` | `set-remove s k` |
| `vec-pop v`, `vec-last v`, `vec-len`, `vec-cap`, `vec-free` | `map-entries`, `map-size`, `map-map-values` | `map-len`, `map-find` | `set-len s` |

## Notes

- `core/map` keys must be Strings (compared with `str-eq`); it is an association list (deterministic, O(n)).
- `vec-get`/`vec-last` out of range return the word `-1` typed as `T`: check `vec-len` first for non-Int elements.
- Not available: `vec-slice`, `vec-append`, `vec-clear`, set union/intersection.

## See Also

- [data-two-map-types](data-two-map-types.md)
- [own-let-copies-word](own-let-copies-word.md) - why buffers alias
