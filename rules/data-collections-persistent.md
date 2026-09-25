# data-collections-persistent

> `Vec` is generic (`(Vec T)`), `core/map` is `(Map String V)`, and `collections/map`/`collections/set` hold `Int`s; every operation returns an updated value: always rebind to the result and treat the old value as used up, because versions share storage.

## Why It Matters

`collections/vec` is `(deftype Vec (VecC (Array T) Int Arena))`: a typed, bounds-checked runtime array, the length this version sees, and the arena the array lives in. `vec-get` returns `T`, so a String element prints and compares as text. Operations return a new `Vec` that **shares the array** with the old one until one of them outgrows it: `vec-set` writes the shared array, and a `vec-push` onto an old version overwrites what a newer version sees at that index. `collections/map` and `collections/set` are Int-keyed arena tables with the same sharing.

The compiler may reuse the old `Vec` record's block for the new one when the old value is provably unique and dead (`compiler/reuse.zyl`, Perceus-style "functional but in place"; `ZYL_REUSE=0` disables it). That is invisible to the program: it never changes what a reachable value holds, and it does not stop the array sharing above.

## Bad

```lisp
(use collections/vec)
(let v (vec-create-default 10)
  (begin
    (vec-push v "a")          ; result discarded: v still has length 0
    (print (vec-len v))))     ; 0

(let v1 (vec-push (vec-create-default 4) 1)
  (let v2 (vec-push v1 2)     ; v1 and v2 share storage
    (let v3 (vec-push v1 99)  ; writes slot 1 of the shared array
      (print v2))))           ; [1, 99]

(vec-create 0 10)             ; E_TYPE_MISMATCH: expected `Arena`, found `Int`
```

## Good

```lisp
(use collections/vec)
(use core/map)
(defn main ()
  (let-mut v (vec-create-default 10)
    (begin
      (set! v (vec-push v "hello"))
      (set! v (vec-push v "world"))
      (print (vec-get v 1))                             ; world
      (print v)                                         ; [hello, world]
      (print (map-insert (map-new) "k" (vec-get v 0)))  ; {k: hello}
      0)))
```

## API cheatsheet

| Vec (`collections/vec`) | Map (`core/map`) | Int map (`collections/map`) | Int set (`collections/set`) |
|---|---|---|---|
| `(vec-create arena cap)` (`arena`: an `Arena`) | `(map-new)` | `(map-create arena cap)` | `(set-create arena cap)` |
| `(vec-create-default cap)` (new private arena) | | `map-create-default cap` | `set-create-default cap` |
| `vec-push v x` (grows by doubling) | `map-insert m k v` (replaces) | `map-put m k v` | `set-add s k` |
| `vec-get v i` (out of range: `E_INDEX_OUT_OF_BOUNDS` panic) | `map-get m k` returns `Option`; `map-get-or m k d` | `map-get m k default` | `set-contains s k` (Bool) |
| `vec-get-or v i default` | `map-has`, `map-remove` | `map-has` (Bool), `map-remove` | `set-remove s k` |
| `vec-set v i x` (`i` = len within cap extends; other out-of-range `i`: unchanged) | `map-entries`, `map-size`, `map-map-values f m` | `map-len`, `map-find` | `set-len s` |
| `vec-pop v` (empty: unchanged), `vec-last v` (empty: panic), `vec-len`, `vec-cap`, `vec-free` | | | |

For a window on a Vec without copying, use `collections/slice` ([data-views-and-slices](data-views-and-slices.md)).

## Notes

- `core/map` keys must be Strings (compared with `str-eq`; an Int key is `E_TYPE_MISMATCH`); it is an association list (deterministic, O(n)).
- The `arena` argument is typed `Arena`: get one from `(arena-create block-size)` (`allocator/allocator`). An Int is rejected at compile time. `*-create-default` makes a new private arena per call that is **never freed**, so calling it in a loop leaks one arena per call.
- `cap` is the initial capacity in elements: 0 is fine (the first push allocates 16), a negative value means 0, and growth copies into a doubled array in the same arena (one `memcpy`; the old array stays until the arena is reset).
- `arena-reset`/`arena-destroy` free every collection built in that arena at once; a collection is valid only while its arena lives. Sending a collection to an actor shares it (not copied). `core/map` needs no arena. See [own-heap-never-freed](own-heap-never-freed.md).
- Not available: `vec-append`, `vec-clear`, `vec-slice` (use `slice-vec`), set union/intersection.

## See Also

- [data-two-map-types](data-two-map-types.md)
- [data-views-and-slices](data-views-and-slices.md)
- [own-let-copies-word](own-let-copies-word.md) - why buffers alias
