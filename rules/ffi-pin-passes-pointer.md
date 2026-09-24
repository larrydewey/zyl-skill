# ffi-pin-passes-pointer

> Use `ffi-pin` only when C expects a **pointer to** a value (out-parameters); pass Ints and Strings directly.

## Why It Matters

`(ffi-pin v)` copies one word into a slot in the non-moving pin arena and returns the **slot's address**. C therefore receives `int64_t *` (for an Int) or `char **` (for a String). Passing `(ffi-pin "text")` to `puts` is a bug: it gets a pointer to the string pointer. `ffi-unpin` returns the word currently in the slot (after C wrote it) and frees nothing; a non-pin pointer prints `zyl: ffi-unpin: pointer not from ffi-pin/Pin arena` and yields 0.

## Bad

```lisp
(ffi-call "puts" (ffi-pin "hello") 1000)      ; puts gets char**
(ffi-call "free" (ffi-pin p) 1000)            ; frees the slot address, not p
```

## Good

```lisp
;; C: int64_t zyl_divmod(int64_t a, int64_t b, int64_t *rem)
(defn print-divmod (a b)
  (let slot (ffi-pin 0)                       ; placeholder slot for the out-param
    (let q (ffi-call "zyl_divmod" a b slot 1000)
      (begin
        (print q)
        (print (ffi-unpin slot))))))          ; remainder written by C
```

## Notes

- Slots are never freed individually; the pin arena (256 KiB blocks) is released at exit. Every `ffi-pin` holds its slot until then.
- A `Secret` must go through `ffi-pin` (`E_FFI_PIN_REQUIRED`); other values need not.
- `stdlib/ffi/ffi.zyl` wrappers (`ffi-pin-value`, `ffi-unpin-value`, `ffi-safe-call`, `ffi-pin-call-unpin`) add no checking.

## See Also

- [ffi-int64-only-no-floats](ffi-int64-only-no-floats.md)
- [secret-five-prohibitions](secret-five-prohibitions.md)
