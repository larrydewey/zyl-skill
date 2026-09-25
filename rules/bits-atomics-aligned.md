# bits-atomics-aligned

> Use `bytebuf-atomic-*` only at 8-aligned offsets that fit in the buffer, and know which ones return old vs new values.

## Why It Matters

Atomics are sequentially consistent operations on 8-byte words. An unaligned or out-of-range offset does **nothing and returns 0** (an unaligned atomic would not be lock-free on x86_64). Return conventions differ per operation, which matters for CAS loops and counters.

| Form | Returns |
|---|---|
| `(bytebuf-atomic-load buf off)` | the word |
| `(bytebuf-atomic-store buf off v)` | 1 |
| `(bytebuf-atomic-add/sub/max/min buf off v)` | the **new** value |
| `(bytebuf-atomic-fetch-add buf off v)` | the **old** value |
| `(bytebuf-atomic-cas buf off expected desired)` | 1 if swapped, else 0 |
| `(align-check ptr alignment)` | 1 if aligned, else 0 (does not raise) |

```lisp
(bytebuf-atomic-store buf 8 41)   ; 1
(bytebuf-atomic-add buf 8 1)      ; 42
(bytebuf-atomic-cas buf 8 42 7)   ; 1, word now 7
(bytebuf-atomic-load buf 3)       ; 0 -- not 8-aligned
```

## Notes

- Address-based atomics are in `atomic/atomic`: `atomic-load`, `atomic-store`, `atomic-add`, `atomic-sub`, `atomic-max`, `atomic-min`, `atomic-cas addr expected new`, `atomic-fetch-add`, `atomic-incr`, `atomic-decr`.
- The handle must be a `ByteBuf`: atomics on a `ByteSlice` (or an `Int`) are `E_TYPE_MISMATCH`. Offsets and values are `Int`; so is every result.
- `align-check` takes a raw address (an `Int`, e.g. from `bytebuf-ptr`), not a handle.
- There is no `TAtomic` type; atomics are operations.
- `E_ATOMIC_ABA` (CAS outside Pin) is not enforced.

## See Also

- [bits-bytebuf-regions](bits-bytebuf-regions.md)
