# own-regions-status

> Write code against the regions that exist today: Stack (frames), one Heap arena, and Pin; Global and Circular are not implemented.

## Why It Matters

The spec's five regions and rules R1–R8 describe a complete static system; the compiler implements a small, safe subset. Knowing which rules are real prevents both over-trusting and fighting the checker.

| Region | Spec purpose | Today |
|---|---|---|
| Stack | non-escaping values | params and `let` locals; one proven ADT shape |
| Heap | escaped values, captures | every other struct/ADT/closure; bump arena, freed at exit |
| Global | immutable constants | **not implemented**: top-level `def` unreadable |
| Circular | cyclic structures | **not implemented** (immutability makes cycles impossible to build anyway) |
| Pin | non-moving FFI memory | `ffi-pin` copies one word into the pin arena |

| Rule | Status |
|---|---|
| R1 stack if no escape | one shape only |
| R2/R5 escape → heap | trivially met (heap by default) |
| R3 Send across actors | syntactic: `let-mut` in `spawn`/`send` is `E_CAPABILITY_LEAK`; Secret is `E_SECRET_ESCAPE` |
| R4 FFI needs Pin + pinnable | pinnability partly checked; Pin required only for `Secret` |
| R6 cycles → Circular, R7 globals | not implemented |
| R8 pin arena never compacted | holds |

`E_REGION_ESCAPE` is catalogued and never raised. Byte-buffer region rules (`E_BYTEBUF_NOT_PIN`, `E_STACK_BYTEBUF_RETURN`, `E_GLOBAL_BYTEBUF_MUT`, `E_ATOMIC_ABA`) are not enforced.

## Notes

- Direct tail calls (≤6 args) are jumps; other deep recursion relies on the big `main` stack (spec §14).
- Capability kinds: see [own-capability-kinds](own-capability-kinds.md).

## See Also

- [own-heap-never-freed](own-heap-never-freed.md)
- [bits-regions-unenforced-use-pin](bits-regions-unenforced-use-pin.md)
