# bits-bounds-fail-closed

> Check the results of byte loads, stores and appends: out-of-range accesses do nothing and return 0 instead of failing.

## Why It Matters

Every access is bounds-checked against the buffer's capacity (or slice length). An out-of-range offset is not UB and not a crash: a load returns 0, a store does nothing and returns 0 (success returns 1), an oversized append returns 0 and writes nothing. That protects memory but can hide offset-arithmetic bugs as silent zeros.

## Good

```lisp
(let buf (bytebuf Heap 64)
  (begin
    (store-u8 :le buf 0 200)       ; 1
    (load-u8 :le buf 0)            ; 200
    (load-i8 :le buf 0)            ; -56
    (load-u8 :le buf 64)           ; 0 -- out of range
    (if (= (store-u8 :le buf 100 1) 0)
      (error "offset out of range")
      0)))
```

## Notes

- `E_BYTE_OOB`, `E_BYTEBUF_CAP_EXCEEDED`, `E_BYTEBUF_OVERLAP`, `E_BYTEBUF_INVALID` are catalogued, never raised.
- A `Secret` may not be a load/store offset (`E_CT_VIOLATION`).

## See Also

- [bits-bytebuf-basics](bits-bytebuf-basics.md)
