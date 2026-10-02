# bits-bytebuf-regions

> Write `Pin` for any buffer whose address you take or share; `Stack` buffers live in the frame region and must not escape, and the other bytebuf region rules are still unchecked.

## Why It Matters

The region argument is **not** part of the type: every buffer is a `ByteBuf`, and the type checker does not distinguish regions (an annotation like `(ByteBuf Stack)` is accepted and ignored). Region inference is what acts on it. A `(bytebuf Stack N)` is really allocated in the function's frame region and released on return; one that is returned, stored, sent, or passed to code that may keep it is a compile-time `E_REGION_ESCAPE`, located at the allocation. Byte loads, stores, appends and atomics are classified, and so are calls to functions whose parameter summary says the argument does not escape, so using the buffer locally or handing it to a helper that only reads it is fine. Every other region (`Heap`, `Global`, `Circular`, `Pin`) still gets the same stable, zero-initialized heap allocation, and these designed rules are not checked (their old codes were removed from the catalog on 2026-10-02):

| Rule | Today |
|---|---|
| `bytebuf-ptr` only on Pin buffers | not checked for other regions; a Stack buffer's `bytebuf-ptr` passed to `ffi-call` is `E_REGION_ESCAPE` |
| Stack buffer may not escape | checked, reported as `E_REGION_ESCAPE` |
| Global buffer immutable | not checked |
| CAS only on Pin memory | not checked |

Writing the correct region now keeps the program valid when the remaining checks land.

## Bad

```lisp
(defn make-buf () (bytebuf Stack 16))            ; E_REGION_ESCAPE: returned
(defn main () (begin (print (bytebuf-cap (make-buf))) 0))
```

```lisp
(capabilities actor)
(defn leak ((tx (Tx ByteSlice)))
  (let b (bytebuf Stack 16)
    (chan-send tx (byteslice b 0 4))))           ; E_REGION_ESCAPE: a slice points into b
(defn main () (begin (leak (chan-tx (chan 1))) 0))
```

## Good

```lisp
(capabilities ffi)
(extern "memset" (Int Int Int) Int :timeout 1000)   ; a foreign call needs its C signature

(defn checksum (n)                               ; Stack buffer used locally: reclaimed on return
  (let b (bytebuf Stack 64)
    (let _ (store-u8 :le b 0 n)
      (load-u8 :le b 0))))

(defn main ()
  (let shared (bytebuf Pin 64)
    (begin
      (ffi-call "memset" (bytebuf-ptr shared) 7 64)
      (print (load-u8 :le shared 63))            ; 7
      (print (checksum 300))                     ; 44: the store keeps the low byte
      0)))
```

## Notes

- Foreign `ffi-call` arguments are always classified as heap, so an abandoned timed-out call never holds a released region; passing a Stack buffer's `bytebuf-ptr` to a foreign function is `E_REGION_ESCAPE`. Use a `Pin` buffer.
- `E_REGION_ESCAPE` comes from region inference, which runs after type checking: `zyl check` passes the Bad examples above, and a build rejects them.
- The filename predates real regions; the rule now covers the Stack check as well.

## See Also

- [own-regions-status](own-regions-status.md)
- [bits-bytebuf-basics](bits-bytebuf-basics.md)
