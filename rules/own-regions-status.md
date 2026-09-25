# own-regions-status

> Regions are real: per-call frame and result regions, the process heap, `with-region` scopes and Pin; Global and Circular are names only.

## Why It Matters

The spec's five regions and rules R1–R8 describe a complete static system. Since 2026-09-24 the compiler infers regions over ICNF and reclaims memory by region; knowing which parts are real prevents both over-trusting and fighting the checker.

| Region | Spec purpose | Today |
|---|---|---|
| Stack | non-escaping values | the call's frame region (L): released on return, before a tail jump, or on unwind; plus the `IStackVariant` rewrite and `(bytebuf Stack N)` |
| Heap | escaped values, captures | the process heap (H), never freed before exit; results go to the caller's region (R) |
| Global | immutable constants | top-level `def`: immutable, initialized once before `main`, heap-allocated (no separate region) |
| Circular | cyclic structures | **not implemented** (immutability makes cycles impossible to build anyway) |
| Pin | non-moving FFI memory | `ffi-pin` copies one word into the pin arena |
| (extension) | region registry | `with-region` `arena`/`fixed` scopes, see [own-with-region](own-with-region.md) |

| Rule | Status |
|---|---|
| R1 stack if no escape | frame region for every site proven L |
| R2/R5 escape → heap | yes; region polymorphism puts results in the caller's region |
| R3 Send across actors | `send` arguments are H; `let-mut` in `spawn`/`send` is `E_CAPABILITY_LEAK`; Secret is `E_SECRET_ESCAPE` |
| R4 FFI needs Pin + pinnable | foreign-call arguments are H; Pin required only for `Secret` |
| R6 cycles → Circular, R7 globals | not implemented |
| R8 pin arena never compacted | holds |

`E_REGION_ESCAPE` is raised (with a location) for a `(bytebuf Stack N)` that is returned, stored, sent or passed to code that may keep it, and for a value allocated inside `with-region` that outlives it. `E_REGION_SPEC` and `E_REGION_EXHAUSTED` belong to `with-region`. The other byte-buffer codes (`E_BYTEBUF_NOT_PIN`, `E_STACK_BYTEBUF_RETURN`, `E_GLOBAL_BYTEBUF_MUT`, `E_ATOMIC_ABA`) are still not raised; a Stack buffer escape reports `E_REGION_ESCAPE` instead of `E_STACK_BYTEBUF_RETURN`.

## Notes

- Inference is whole-program and field-insensitive (union-find classes); it over-approximates, so a missed case costs heap memory, never a dangling pointer.
- A call through a function value passes its arguments as H; runtime functions are trusted only from the compiler's table (`rg-ffi-kind`).
- The REPL interpreter ignores regions and `with-region` limits.
- `try`/`catch` frames (`zyl_try_push`) are still `malloc`ed per `try` and never freed.
- Two later mechanisms reduce memory further without changing regions: a self tail call recycles its frame region in place (`zyl_region_recycle`), and the reuse pass (`compiler/reuse.zyl`, `ZYL_REUSE=0` to disable) writes an update of a unique, dead value into the old block. Both are native-backend only; see [own-heap-never-freed](own-heap-never-freed.md).
- `(Pin a)` is the type of a pinned slot; see [ffi-pin-passes-pointer](ffi-pin-passes-pointer.md).
- Tail calls are jumps when their stack arguments fit the caller's incoming area; tail-call arguments are at least R because the frame is gone when the callee runs. See [fn-no-named-let-or-early-return](fn-no-named-let-or-early-return.md).
- Capability kinds: see [own-capability-kinds](own-capability-kinds.md).

## See Also

- [own-heap-never-freed](own-heap-never-freed.md)
- [own-with-region](own-with-region.md)
- [bits-bytebuf-regions](bits-bytebuf-regions.md)
