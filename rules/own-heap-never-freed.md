# own-heap-never-freed

> Expect heap values to live until process exit; in long-running or allocation-heavy code, manage memory with an explicit arena from `allocator/allocator`.

## Why It Matters

Every struct, ADT value and capturing closure not proven local comes from `zyl_heap_alloc`, a bump allocator over one process-wide arena that is **never reset or freed** during the run. There is no GC and no region reclamation yet. A loop that allocates without bound grows until it hits the memory budget: `E_OUT_OF_MEMORY` (budget = `ZYL_MAX_MEMORY` bytes if set, `0` disables it; otherwise 80% of available memory).

## Good

```lisp
(use allocator/allocator)
(defn process-batch (items)
  (let arena (arena-create 1048576)       ; block size in bytes
    (let buf (arena-alloc-zeroed arena 4096)
      (let result (work arena buf items)
        (begin
          (arena-destroy arena)           ; bulk release
          result)))))
```

## Allocator API

`(arena-create block-size)`, `(arena-alloc a n)`, `(arena-alloc-zeroed a n)`, `(arena-reset a)`, `(arena-destroy a)`, `(arena-used a)`, `(arena-capacity a)`, `(alloc-malloc n)`, `(alloc-free p)`, `(alloc-read-int addr)`, `(alloc-write-int addr v)`, `(str-intern arena s)`, `(buf-append dst src)`.

## Notes

- `(arena-create block-size)`: `block-size` below 16 means the 64 KiB default; returns 0 if memory is exhausted. `arena-reset` frees all blocks (arena stays usable); `arena-destroy` frees everything (handle dead).
- Collections (`vec-create`/`map-create`/`set-create arena cap`) take a handle from `arena-create`, or any value <= 0 for a new private arena that is never freed (leaks one arena per call inside a loop). Any other positive `Int` is not an arena: unchecked, crashes. Resetting/destroying an arena frees every collection in it at once; don't use them afterwards.
- Release collections with `arena-reset`/`arena-destroy`, not `vec-free`/`map-free`: those pass the arena-owned buffer to C `free()` (`zyl_mem_free`), which is undefined for arena memory.
- Only functions taking an arena argument allocate from one: ADT and struct values (`(Some 1)`, `(make-Point 1 2)`) always use the runtime heap above.
- Arena memory is raw: values you build there are `Int` addresses to Zyl, not typed values.
- The compiler itself allocates everything from one 1 GiB arena per compile.
- Stack promotion happens for exactly one shape (see [own-stack-promotion](own-stack-promotion.md)); a missed case costs an allocation, never a dangling pointer.

## See Also

- [proj-buf-append-appends](proj-buf-append-appends.md)
- [own-regions-status](own-regions-status.md)
