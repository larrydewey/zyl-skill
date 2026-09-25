# bits-bounds-fail-closed

> Check the results of byte loads, stores and appends: out-of-range accesses do nothing and return 0 instead of failing.

## Why It Matters

Every access is bounds-checked against the buffer's capacity (or slice length). An out-of-range offset is not UB and not a crash: a load returns 0, a store does nothing and returns 0 (success returns 1), an oversized append returns 0 and writes nothing. That protects memory but can hide offset-arithmetic bugs as silent zeros.

## Good

```lisp
(defn main ()
  (let buf (bytebuf Heap 64)
    (begin
      (store-u8 :le buf 0 200)       ; 1
      (load-u8 :le buf 0)            ; 200
      (load-i8 :le buf 0)            ; -56
      (load-u8 :le buf 64)           ; 0 -- out of range
      (if (= (store-u8 :le buf 100 1) 0)
        (error "offset out of range")  ; taken: PANIC: offset out of range
        0))))
```

## Notes

- The native backend inlines one-byte loads and stores (a bound check, then the access; when the handle is a parameter that never changes, its data pointer and bound are loaded once per function). The inline path gives exactly the runtime's results: anything the runtime would reject, including an invalid handle, reads 0 or stores nothing. Wider accesses call the runtime under the same rule.
- Only the handle is typed (`ByteBuf`/`ByteSlice`); the offset is an ordinary `Int`, so the type checker cannot catch an out-of-range offset.
- `E_BYTE_OOB`, `E_BYTEBUF_CAP_EXCEEDED`, `E_BYTEBUF_OVERLAP`, `E_BYTEBUF_INVALID` are catalogued, never raised.
- A `Secret` may not be a load/store offset (`E_CT_VIOLATION`).

## See Also

- [bits-bytebuf-basics](bits-bytebuf-basics.md)
- [bits-wide-loads-and-stores](bits-wide-loads-and-stores.md)
