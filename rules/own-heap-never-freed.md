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

- Collections take an arena argument; `0` creates a private one. `vec-free`/`map-free` release them.
- Arena memory is raw: values you build there are `Int` addresses to Zyl, not typed values.
- The compiler itself allocates everything from one 1 GiB arena per compile.
- Stack promotion happens for exactly one shape (see [own-stack-promotion](own-stack-promotion.md)); a missed case costs an allocation, never a dangling pointer.

## See Also

- [proj-buf-append-appends](proj-buf-append-appends.md)
- [own-regions-status](own-regions-status.md)
