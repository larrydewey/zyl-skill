# own-heap-never-freed

> Values that escape into the heap live until process exit; per-call regions reclaim everything the compiler proves short-lived and the reuse pass updates unique dead values in place, so for bulk temporaries use `with-region`, or an `allocator/allocator` arena for raw memory.

## Why It Matters

Region inference (`region_inference.zyl`, since 2026-09-24) gives every allocation site one of three levels:

| Level | Where it goes | Released |
|---|---|---|
| L (frame) | the call's own region | on return, before a tail jump, or when a caught panic / failing test unwinds it |
| R (result) | the region the caller chose for the result | with the caller's region |
| H (heap) | the process heap (`zyl_heap_alloc`) | **never**, until exit |

A value is H when it escapes in a way the compiler does not track: stored in a global, sent to an actor, handed to a foreign `ffi-call`, captured by a closure called through a function value, or passed to a runtime function not in the compiler's region-safe table (`rg-ffi-kind`). H memory is a bump allocator that is never reset (the only exception is in-place reuse, below); a loop that keeps escaping values grows until `E_OUT_OF_MEMORY` (budget = `ZYL_MAX_MEMORY` bytes if set, `0` disables it; otherwise 80% of available memory).

Temporaries that die inside the call are reclaimed: a program that built and dropped 20000 lists went from a 630 MB to a 12 MB peak. A self tail call (a loop) recycles the frame region: it keeps the region's first block and empties it rather than releasing and reopening it (`zyl_region_recycle`, native backend).

**In-place reuse** (`compiler/reuse.zyl`, Perceus-style, decided statically): when an update such as `(vec-push v x)`, a struct rebuilt with one field changed, or a list rebuilt cell by cell consumes a value that is provably **unique** (bound to a fresh value, never aliased, stored, captured, or passed where it may escape) and **dead** (not used afterwards, and not bound outside the loop the update is in), the new record is written into the old one's block instead of allocating. It applies in every level, heap included, so a loop that threads one vector through `vec-push` no longer grows memory per step (bench: 718 to 260 MB). Nothing can observe it; the old value is unreachable. Only the native backend honours it; `ZYL_REUSE=0` turns it off and `ZYL_REUSE_DEBUG=1` prints each function's reuse facts at compile time.

## Bad

```lisp
(def seen (vec-create-default 16))        ; global: everything pushed here is heap
(defn step (i)
  (vec-push seen (Some i)))               ; each (Some i) escapes: kept to exit
```

## Good

```lisp
(defn total (n)
  (wsum (build n (WN))))                  ; the list dies in this call: frame region

(defn bounded (n)                         ; explicit, capped arena for bulk work
  (with-region (arena :block 65536 :limit 1048576)
    (wsum (build n (WN)))))               ; only the Int escapes
```

For raw memory (bytes addressed as `Ptr`), use an explicit arena:

```lisp
(use allocator/allocator)
(let arena (arena-create 1048576)
  (let result (work arena items)
    (begin (arena-destroy arena) result)))
```

## Allocator API

`(arena-create block-size)`, `(arena-alloc a n)`, `(arena-alloc-zeroed a n)`, `(arena-reset a)`, `(arena-destroy a)`, `(arena-used a)`, `(arena-capacity a)`, `(alloc-malloc n)`, `(alloc-free p)`, `(alloc-read-int addr)`, `(alloc-write-int addr v)`, `(str-intern arena s)`, `(buf-append dst src)`.

## Notes

- Classes are field-insensitive: a variant, its fields and anything read out of it share one level, so one escaping use keeps the whole structure in the heap.
- `(arena-create block-size)` gives an `Arena` handle: `block-size` below 16 means the 64 KiB default. `arena-alloc`/`arena-alloc-zeroed` give a `Ptr`. `arena-reset` frees all blocks; `arena-destroy` frees everything (handle dead).
- Collections take a typed `Arena`: `(vec-create arena cap)`, `map-create`, `set-create` with a handle from `arena-create` (passing `0` is `E_TYPE_MISMATCH`), or `vec-create-default`/`map-create-default`/`set-create-default cap` for a new private arena that is never freed (one arena per call inside a loop). Release arena-backed collections with `arena-reset`/`arena-destroy`, not `vec-free`/`map-free`.
- Arena memory is raw: what you get is a `Ptr`, read and written with `alloc-read-int`/`alloc-write-int`, not typed Zyl values.
- The compiler itself gains little from regions (its build time and peak memory are unchanged): its data mostly escapes into results or global tables.
- `ZYL_REGIONS=0` at compile time makes every site H (the old behaviour), for bisecting a suspected region bug.

## See Also

- [own-with-region](own-with-region.md)
- [own-regions-status](own-regions-status.md)
- [proj-buf-append-appends](proj-buf-append-appends.md)
