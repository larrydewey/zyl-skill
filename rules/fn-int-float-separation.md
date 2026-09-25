# fn-int-float-separation

> Never mix `Int` and `Float` in one operation: it is `E_TYPE_MISMATCH`, there is no implicit conversion, and there are no conversion built-ins.

## Why It Matters

Arithmetic (`+ - * / %`) takes two values of one type from the closed Num class {Int, Float}. Since 2026-09-25 mixing is a compile error wherever it happens, including through inference: `(defn half (n) (* n 0.5))` makes `n` a Float, so `(half 3)` is rejected at the call. (Before, `(+ 1 2.5)` compiled and computed garbage.) Write Float literals with a `.`: `2.0`, not `2`.

There are no `float`/`int` built-ins: `(float 42)` is `E_UNBOUND_VARIABLE`. The one conversion available to user code is the runtime entry `(ffi-call "zyl_f_of_int" n 1000)` (Int to Float); Float to Int truncation has no typed entry (`zyl_f_to_int` is untyped, so calling it is `E_CANNOT_INFER`).

## Bad

```lisp
(+ 1 2.5)                         ; E_TYPE_MISMATCH: cannot unify Float with Int
(defn half (n) (* n 0.5))
(half 3)                          ; E_TYPE_MISMATCH at the call
(float 42)                        ; E_UNBOUND_VARIABLE
```

## Good

```lisp
(+ 1.0 2.5)                       ; 3.500000
(defn scale ((x Float)) (* x 0.5))
(* (ffi-call "zyl_f_of_int" 42 1000) 0.5)   ; 21.000000
;; Keep integer math in Int (e.g. fixed-point: value * 1000)
```

## Notes

- Float arithmetic is IEEE-754 binary64 with separate SSE2 multiply/add (no FMA), so results are bit-reproducible in practice.
- Float division by zero gives `inf`/`nan` (no trap); `print` shows six decimals. Unary minus on a Float is a Float negation.
- Comparison and ordering also require both sides to have one type.
- An `extern` may not use `Float` (the timed FFI worker passes machine words); runtime entries typed with Float work. See [ffi-extern-word-sized-types](ffi-extern-word-sized-types.md).
- Float constants are not constant-folded, and functions doing Float arithmetic are compiled by the stack-machine backend rather than the register-allocated one.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md) - every value has one static type
- [type-sound-checking](type-sound-checking.md) - what the checker rejects
