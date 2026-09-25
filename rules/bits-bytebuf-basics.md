# bits-bytebuf-basics

> Allocate packed bytes with `(bytebuf Region N)` using literal region and capacity; pass the handle as a `ByteBuf` or `ByteSlice`, annotating parameters the checker cannot settle.

## Why It Matters

`bytebuf` gives a fixed-capacity, zero-initialized block. Both arguments are compile-time literals: region ∈ `Stack Heap Global Circular Pin`, capacity an integer literal (anything else is `E_UNEXPECTED_TOKEN_IN_EXPR`). The handles are their own types: `bytebuf` returns a `ByteBuf`, `byteslice`/`byteslice-sub` return a `ByteSlice`, and passing an `Int` (or any other type) where a handle is expected is a compile-time `E_TYPE_MISMATCH`. Loads and stores accept either handle, so a parameter used only for loads and stores has no single type: if nothing else in its function group settles it, it is `E_CANNOT_INFER` ("cannot tell whether this is a ByteBuf or a ByteSlice") and needs an annotation.

## Bad

```lisp
(defn first-byte (b) (load-u8 :le b 0))     ; E_CANNOT_INFER: ByteBuf or ByteSlice?
(load-u8 :le 5 0)                           ; E_TYPE_MISMATCH: expected `ByteBuf`, found `Int`
(byte 300)                                  ; E_BYTE_VALUE_OOB
```

## Good

```lisp
(defn first-byte ((b ByteBuf)) (load-u8 :le b 0))
(defn first-of (s) (load-u8 :le (byteslice-sub s 0 1) 0))   ; byteslice-sub fixes s as a ByteSlice

(defn main ()
  (let b (bytebuf Heap 16)
    (let _ (store-u8 :le b 0 255)
      (begin
        (print (load-u8 :le b 0))    ; 255 (zero-extended)
        (print (load-i8 :le b 0))    ; -1  (sign-extended)
        (print (bytebuf-cap b))      ; 16
        (print (bytebuf-len b))      ; 0: bytes appended so far
        0))))
```

## Family

| Form | Handle | Notes |
|---|---|---|
| `(byte n)` | | literal 0..255 (else `E_BYTE_VALUE_OOB`); an `Int` |
| `(bytebuf Region cap)` | returns `ByteBuf` | zero-initialized |
| `bytebuf-cap`, `bytebuf-len`, `bytebuf-ptr` | `ByteBuf` | ptr: stable raw address (an `Int`), the FFI hatch |
| `(byteslice buf off len)` | `ByteBuf` in, `ByteSlice` out | zero-copy view, bounds-checked at creation |
| `(byteslice-sub s off len)` | `ByteSlice` in and out | |
| `(bytebuf-append dst slice)` | `ByteBuf`, `ByteSlice` | appends a **slice**, fails closed if it would exceed capacity; memmove-safe |
| `load-u8`/`load-i8`/`store-u8`/`store-i8` and the wide forms | either | `(load-u8 :le buf off)`, `(store-u8 :le buf off v)` |
| `bytebuf-atomic-*` | `ByteBuf` | see [bits-atomics-aligned](bits-atomics-aligned.md) |

## Notes

- The type does not record the region: an annotation such as `(ByteBuf Stack)` is accepted but means nothing more than `ByteBuf`, and the checker does not distinguish buffers by region.
- Operands are evaluated left to right, buffer first, then offset, then value.
- No stdlib module uses byte buffers yet: `stdlib/math` represents byte strings as `Words` arrays with one byte per word.
- A `Stack` buffer lives in the frame region and must not escape (`E_REGION_ESCAPE`); every other region gets the same heap allocation (see [bits-bytebuf-regions](bits-bytebuf-regions.md)).

## See Also

- [bits-bounds-fail-closed](bits-bounds-fail-closed.md)
- [bits-wide-loads-and-stores](bits-wide-loads-and-stores.md)
