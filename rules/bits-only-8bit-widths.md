# bits-only-8bit-widths

> Use `load-u16`..`load-u64`, `load-i16`..`load-i64` and `store-*` for wide values, with an explicit `:le`/`:be`; buffers are typed `ByteBuf`/`ByteSlice`.

## Why It Matters

Until 2026-09-24 only the 8-bit forms existed and the wider names were rejected with `E_RESERVED_KEYWORD`, so older code builds wide values from byte loads and shifts. The wide forms now work: same argument shape as the 8-bit ones, the selector picks byte order, signed loads sign-extend from the loaded width, and the bounds check covers the whole width (a partly out-of-range access reads 0 or stores nothing, returning 0). `load-u64` returns the bit pattern as an `Int`.

A handle is typed: `bytebuf` returns `ByteBuf`, `byteslice`/`byteslice-sub` return `ByteSlice`, and passing an `Int`/`Float`/`Bool`/`String` where a handle is expected is `E_TYPE_MISMATCH`.

## Bad

```lisp
(defn load-u32-le (buf off)                    ; hand-rolled, four calls
  (bit-or (load-u8 :le buf off)
          (shl (load-u8 :le buf (+ off 1)) 8)
          (shl (load-u8 :le buf (+ off 2)) 16)
          (shl (load-u8 :le buf (+ off 3)) 24)))
(load-u16 :le 7 0)                             ; E_TYPE_MISMATCH: expected ByteBuf
```

## Good

```lisp
(defn header-len ((b ByteBuf)) (load-u32 :be b 4))
(store-i64 :le buf 8 -2)
(load-i16 :le buf 8)                           ; -2
```

## Notes

- Implementation: the width rides in the `Endian` value (`EWide`), lowered to `zyl_load_n`/`zyl_load_n_signed`/`zyl_store_n`.

## See Also

- [bits-bytebuf-basics](bits-bytebuf-basics.md)
- [bits-bounds-fail-closed](bits-bounds-fail-closed.md)
