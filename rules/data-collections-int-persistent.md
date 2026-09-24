# data-collections-int-persistent

> `Vec`, `Map` and `Set` hold `Int`s only and return an updated value: always rebind to the result and treat the old value as used up.

## Why It Matters

`collections/vec`, `collections/map` and `collections/set` are arena-backed structs `(ptr len cap arena)`. Operations return a new struct that **shares the buffer** with the old one. `vec-push` onto an old version, `map-put` on an existing key, and `map-remove`/`set-remove` (which compact in place) are all visible through older versions. Out-of-range reads return sentinels, not errors.

## Bad

```lisp
(use collections/vec)
(let v (vec-create 0 10)
  (begin
    (vec-push v 42)          ; result discarded: v still has length 0
    (print (vec-len v))))    ; 0

(let v1 (vec-push (vec-create 0 4) 1)
  (let v2 (vec-push v1 2)    ; v1 and v2 share storage
    (vec-push v1 99)))       ; overwrites v2's element 1
```

## Good

```lisp
(use collections/vec)
(defn main ()
  (let-mut v (vec-create 0 10)
    (begin
      (set! v (vec-push v 42))
      (set! v (vec-push v 99))
      (print (vec-len v))     ; 2
      (print (vec-get v 0))   ; 42
      (print (vec-get v 5))   ; -1 (out of bounds sentinel)
      0)))
```

## API cheatsheet

| Vec (`collections/vec`) | Map (`collections/map`) | Set (`collections/set`) |
|---|---|---|
| `(vec-create arena cap)` (arena 0 = private) | `(map-create arena cap)` | `(set-create arena cap)` |
| `vec-create-default cap` | `map-create-default cap` | |
| `vec-push v x` (grows) | `map-put m k v` (overwrite in place) | `set-add s k` (dupes ignored) |
| `vec-get v i` (-1 OOB) | `map-get m k default` | `set-contains s k` |
| `vec-set v i x` (past len within cap extends; beyond cap no-op) | `map-has m k` | `set-remove s k` |
| `vec-pop v`, `vec-last v` (-1 empty) | `map-remove m k` | `set-len s` |
| `vec-len`, `vec-cap`, `vec-free` | `map-len`, `map-cap`, `map-find`, `map-free` | `set-cap`, `set-find` |

## Notes

- `-1` sentinels are ambiguous if `-1` is a legitimate element: check `vec-len` first.
- Not available: `vec-slice`, `vec-append`, `vec-clear`, `map-keys`/`map-values`/`map-entries`, set union/intersection.
- Lookup is a linear scan in insertion order: deterministic, fine for small maps.
- For any value type, use `List` or `collections/collections` association lists (`assoc-put k v m`, `assoc-get k default m`).

## See Also

- [data-two-map-types](data-two-map-types.md)
- [own-let-copies-word](own-let-copies-word.md) - why buffers alias
