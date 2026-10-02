# data-collections-persistent

> `Vec` is generic (`(Vec T)`), `core/map` is `(Map String V)`, and `collections/intmap`/`collections/set` hold `Int`s; every operation returns an updated value: always rebind to the result and treat the old value as used up, because versions share storage.

## Why It Matters

`collections/vec` is `(deftype Vec (VecC (Array T) Int))`: a typed, bounds-checked runtime array and the length this version sees. `vec-get!` returns `T`, so a String element prints and compares as text. Operations return a new `Vec` that **shares the array** with the old one until one of them outgrows it: `vec-set!` writes the shared array, and a `vec-push` onto an old version overwrites what a newer version sees at that index. `collections/intmap` and `collections/set` are Int-keyed tables with the same sharing.

Storage comes from the region the allocating call runs in: a Vec that does not outlive its frame is reclaimed on return, and one that does goes to the caller's region or the heap. There is no `Arena` argument any more.

The compiler may reuse the old `Vec` record's block for the new one when the old value is provably unique and dead (`compiler/reuse.zyl`, Perceus-style "functional but in place"; `ZYL_REUSE=0` disables it). That is invisible to the program: it never changes what a reachable value holds, and it does not stop the array sharing above.

## Bad

```lisp
(use collections/vec)
(defn main ()
  (begin
    (let v (vec-new)
      (begin
        (vec-push v "a")            ; result discarded: v still has length 0
        (print (vec-len v))))       ; 0
    (let v1 (vec-push (vec-new-cap 4) 1)
      (let v2 (vec-push v1 2)       ; v1 and v2 share storage
        (let _v3 (vec-push v1 99)   ; writes slot 1 of the shared array
          (print v2))))             ; [1, 99]
    0))
```

## Good

```lisp
(use collections/vec)
(use core/map)
(defn main ()
  (let-mut v (vec-new)
    (begin
      (set! v (vec-push v "hello"))
      (set! v (vec-push v "world"))
      (print (vec-get! v 1))                             ; world
      (print v)                                         ; [hello, world]
      (print (map-insert (map-new) "k" (vec-get! v 0)))  ; {k: hello}
      (match (vec-get? v 5)
        (Some s (print s))
        (None (print "no element 5")))                  ; no element 5
      0)))
```

## API cheatsheet

| Vec (`collections/vec`) | Map (`core/map`) | Int map (`collections/intmap`) | Int set (`collections/set`) |
|---|---|---|---|
| `(vec-new)`, `(vec-new-cap cap)` | `(map-new)` | `(intmap-new cap)` | `(set-create cap)` |
| `vec-push v x` (grows by doubling) | `map-insert m k v` (replaces) | `intmap-put m k v` | `set-add s k` |
| `vec-get! v i` (out of range: `E_INDEX_OUT_OF_BOUNDS` panic), `vec-get? v i` (`Option`) | `map-get m k` returns `Option`; `map-get-or m k d` | `intmap-get m k default` | `set-contains s k` (Bool) |
| `vec-get-or v i default` | `map-has`, `map-remove` | `intmap-has` (Bool), `intmap-remove` | `set-remove s k` |
| `vec-set! v i x` / `vec-set? v i x` (`i` = len appends; any other out-of-range `i` panics / gives `None`) | `map-entries`, `map-size`, `map-map-values f m` | `intmap-len`, `intmap-find` | `set-len s` |
| `vec-pop v` (empty: unchanged), `vec-last! v` (empty: panic), `vec-last? v`, `vec-len`, `vec-cap`, `vec-free` | | | |

There is no plain `vec-get`, `vec-set` or `vec-last`: a partial operation is spelled `!` (panics) or `?` (returns an `Option`) ([err-partial-ops-spelled](err-partial-ops-spelled.md)). For a window on a Vec without copying, use `collections/slice` ([data-views-and-slices](data-views-and-slices.md)).

## Notes

- `core/map` keys must be Strings (compared with `str-eq`; an Int key is `E_TYPE_MISMATCH`); it is an association list (deterministic, O(n)).
- `cap` is the initial capacity in elements, a hint and not a limit: 0 is fine (the first push allocates 16), a negative value means 0, and growth copies into a doubled array.
- Sending a collection to an actor shares it (not copied). See [own-heap-never-freed](own-heap-never-freed.md).
- Not available: `vec-append`, `vec-clear`, `vec-slice` (use `slice-vec!`/`slice-vec?`), set union/intersection.

## See Also

- [data-two-map-types](data-two-map-types.md)
- [data-views-and-slices](data-views-and-slices.md)
- [err-partial-ops-spelled](err-partial-ops-spelled.md)
- [own-let-copies-word](own-let-copies-word.md) - why buffers alias
