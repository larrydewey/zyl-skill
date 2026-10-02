# own-heap-never-freed

> Values that escape into the heap live until process exit; per-call regions reclaim everything the compiler proves short-lived and the reuse pass updates unique dead values in place, so for bulk temporaries use `with-region` (programs have no raw-memory arena).

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
(deftype WList (WC Int WList) (WN))
(defn build (n acc) (if (= n 0) acc (build (- n 1) (WC n acc))))
(defn wlen (l) (match l (WN 0) (WC _ r (+ 1 (wlen r)))))

(def table (build 100000 (WN)))         ; a top-level def is heap: these cells live until exit

(defn main () (begin (print (wlen table)) 0))
```

## Good

```lisp
(deftype WList (WC Int WList) (WN))
(defn build (n acc) (if (= n 0) acc (build (- n 1) (WC n acc))))
(defn wsum (l) (match l (WN 0) (WC x r (+ x (wsum r)))))

(defn total (n)
  (wsum (build n (WN))))                  ; the list dies in this call: frame region

(defn bounded (n)                         ; explicit, capped arena for bulk work
  (with-region (arena :block 65536 :limit 1048576)
    (wsum (build n (WN)))))               ; only the Int escapes

(defn loop (i acc) (if (= i 0) acc (loop (- i 1) (+ acc (total 1000)))))

(defn main () (begin (print (loop 20000 0)) (print (bounded 100)) 0))
```

Built 20000 times, the 1000-cell list in `total` peaks at 12 MB; compiled with `ZYL_REGIONS=0`, which sends every cell to the heap, the same program peaks at 630 MB.

## Raw memory

There is none for programs. `allocator/allocator` (`arena-create`, `alloc-malloc`, `alloc-read-int`, ...) is for the compiler and the standard library: a program that calls `arena-create` is `E_FFI_RESTRICTED` ("is a raw-memory entry: it is for the compiler, not for programs"). Use `with-region` for bulk temporaries and `bytebuf` for bytes.

## Notes

- Classes are field-insensitive: a variant, its fields and anything read out of it share one level, so one escaping use keeps the whole structure in the heap.
- Collections take no arena: `vec-new`, `vec-new-cap cap`, `intmap-new cap` and `set-create cap` allocate in the region the calling frame runs in, so a collection that does not outlive its frame is reclaimed on return. `vec-free` and `intmap-free` release nothing; they return an empty value, and the old storage goes with its region.
- The compiler itself gains little from regions (its build time and peak memory are unchanged): its data mostly escapes into results or global tables.
- `ZYL_REGIONS=0` at compile time makes every site H (the old behaviour), for bisecting a suspected region bug.

## See Also

- [own-with-region](own-with-region.md)
- [own-regions-status](own-regions-status.md)
- [proj-buf-append-appends](proj-buf-append-appends.md)
