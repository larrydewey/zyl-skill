# ffi-pin-passes-pointer

> Use `ffi-pin` only when C expects a **pointer to** a value (out-parameters): it gives a `(Pin a)`, the address of a slot, which an extern must declare as `(Pin a)`. Pass Ints and Strings directly.

## Why It Matters

`(ffi-pin v)` copies `v`'s one word into a slot in the non-moving pin arena and returns the **slot's address**, typed `(Pin a)` when `v : a`. C therefore receives `int64_t *` (for an Int) or `char **` (for a String). `(ffi-unpin p)` requires `p : (Pin a)` and gives the `a` currently in the slot (after C wrote it); it frees nothing. Because the pin has its own type, the checker catches the classic mistakes: a `(Pin String)` passed where the extern says `String` is `E_TYPE_MISMATCH`, and so is `(ffi-unpin 7)`. It cannot catch an extern that itself declares the wrong thing.

## Bad

```lisp
(capabilities ffi)
(extern "puts" (String) Int)
(defn main () (begin (ffi-call "puts" (ffi-pin "hello") 1000) 0))   ; E_TYPE_MISMATCH: (Pin String) with String
```

```lisp
(capabilities ffi)
(extern "puts" ((Pin String)) Int)         ; type-checks, but puts gets a char** and prints garbage
(defn main () (begin (ffi-call "puts" (ffi-pin "hello") 1000) 0))
```

```lisp
(capabilities ffi)
(defn main () (let _p (ffi-pin (fn (x) x)) 0))   ; E_FFI_TYPE_NOT_PINNABLE: a function cannot be pinned
```

```lisp
(capabilities ffi)
(defn main () (let p (ffi-pin 1) (begin (print (ffi-unpin p 3)) 0)))   ; E_MALFORMED_FORM: exactly one operand
```

## Good

```c
int64_t my_divmod(int64_t a, int64_t b, int64_t *rem) { *rem = a % b; return a / b; }
```

```lisp
(capabilities ffi)
(extern "my_divmod" (Int Int (Pin Int)) Int :timeout 1000)
(defn print-divmod (a b)
  (let slot (ffi-pin 0)                       ; placeholder slot for the out-param
    (let q (ffi-call "my_divmod" a b slot)
      (begin
        (print q)                             ; (print-divmod 17 5): 3
        (print (ffi-unpin slot))))))          ; 2, written by C
(defn main () (begin (print-divmod 17 5) 0))
```

## Notes

- Slots are never freed individually; the pin arena (256 KiB blocks) is released at exit. Every `ffi-pin` holds its slot until then.
- A `Secret` argument to a foreign call must go through `ffi-pin` (`E_FFI_PIN_REQUIRED`); other values need not.
- `stdlib/ffi/ffi.zyl` wrappers: `ffi-pin-value` and `ffi-unpin-value` are the forms as functions (same types); `ffi-safe-call` and `ffi-pin-call-unpin` only wrap an already-computed result in `Ok`, so they catch nothing the call raised before they were entered.
- `ffi-pin` and `ffi-unpin` need the `ffi` capability, in a package and in a lone file (`(capabilities ffi)`).

## See Also

- [ffi-extern-word-sized-types](ffi-extern-word-sized-types.md)
- [ffi-extern-required](ffi-extern-required.md)
- [secret-five-prohibitions](secret-five-prohibitions.md)
