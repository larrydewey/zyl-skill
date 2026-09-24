# bits-bytebuf-basics

> Allocate packed bytes with `(bytebuf Region N)` using literal region and capacity; keep buffer handles in their own bindings.

## Why It Matters

`bytebuf` gives a fixed-capacity, zero-initialized block. Both arguments are compile-time literals: region ∈ `Stack Heap Global Circular Pin`, capacity an integer literal. A buffer handle is an `Int` to the type system: passing an ordinary number where a buffer is expected is not a type error, and because each entry point dereferences the handle to check its magic word, a small integer such as 5 **crashes** the program (0 is treated as "no buffer").

## Good

```lisp
(defn main ()
  (let b (bytebuf Heap 16)
    (let _ (store-u8 :le b 0 255)
      (begin
        (print (load-u8 :le b 0))    ; 255 (zero-extended)
        (print (load-i8 :le b 0))    ; -1  (sign-extended)
        (print (bytebuf-cap b))      ; 16
        (print (bytebuf-len b))      ; bytes appended so far
        0))))
```

## Family

| Form | Notes |
|---|---|
| `(byte n)` | literal 0..255 (else `E_BYTE_VALUE_OOB`) |
| `(bytebuf Region cap)` | zero-initialized |
| `bytebuf-cap`, `bytebuf-len`, `bytebuf-ptr` | ptr: stable raw address, the FFI hatch |
| `(byteslice buf off len)`, `(byteslice-sub s off len)` | zero-copy views, bounds-checked at creation |
| `(bytebuf-append dst slice)` | appends a **slice**, fails closed if it would exceed capacity; memmove-safe |
| `load-u8`/`load-i8`/`store-u8`/`store-i8` | `(load-u8 :le buf off)`, `(store-u8 :le buf off v)` |
| `bytebuf-atomic-*`, `align-check` | see [bits-atomics-aligned](bits-atomics-aligned.md) |

## Notes

- No stdlib module uses byte buffers yet: `stdlib/math` represents byte strings as one byte per word.
- The runtime allocates every region the same way (see [bits-regions-unenforced-use-pin](bits-regions-unenforced-use-pin.md)).

## See Also

- [bits-bounds-fail-closed](bits-bounds-fail-closed.md)
- [bits-only-8bit-widths](bits-only-8bit-widths.md)
