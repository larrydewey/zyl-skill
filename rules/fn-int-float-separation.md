# fn-int-float-separation

> Never mix `Int` and `Float` in one operation; there is no conversion and no diagnostic.

## Why It Matters

`(+ 1 2.5)` is neither rejected nor converted: it computes a wrong answer (`(+ 1.5 2)` evaluates to `1.5`). There are no conversion built-ins: `(float 42)` and `(int 3.7)` fail at link time as undefined functions. The type checker does not catch mixing (it rejects no type errors at all).

## Bad

```lisp
(+ 1 2.5)          ; garbage
(* n 0.5)          ; n is an Int: garbage
(float 42)         ; undefined reference at link time
```

## Good

```lisp
(+ 1.0 2.5)                       ; 3.500000
(defn scale ((x Float)) (* x 0.5))
;; Keep integer math in Int (e.g. fixed-point: value * 1000)
```

## Notes

- Float arithmetic is IEEE-754 binary64 with separate SSE2 multiply/add (no FMA), so results are bit-reproducible in practice.
- Float division by zero gives inf/NaN; `print` shows six decimals.
- Floats do not cross the FFI correctly (see [ffi-int64-only-no-floats](ffi-int64-only-no-floats.md)).
- Float constants are not constant-folded.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md) - how Float operands are recognized
- [type-inference-does-not-reject](type-inference-does-not-reject.md) - no type errors
