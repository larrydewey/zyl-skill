# match-arm-complex

> In a match arm, never combine a constant with two or more calls in one arithmetic expression; bind the calls with `let` first.

## Why It Matters

An arm body shaped like `(+ 1 (f n m) (g n))` is rejected at compile time with `E_MATCH_ARM_COMPLEX` — a guard against a shape older stage binaries miscompiled (it computed 0). The broader old rule ("any binop over two calls computes 0") no longer reproduces, but the compiler source still follows it, and short `let` chains keep code clear of the checked shape.

## Bad

```lisp
(match t
  (Node v l r (+ 1 (size l) (size r))))   ; E_MATCH_ARM_COMPLEX
```

## Good

```lisp
(match t
  (Node v l r
    (let a (size l)
      (let b (size r)
        (+ 1 a b)))))

;; or nest through a two-argument helper
(defn add2 (a b) (+ a b))
(match t (Node v l r (add2 1 (add2 (size l) (size r)))))
```

## Notes

- N-ary arithmetic folds left: `(+ a b c)` is `(+ (+ a b) c)`.
- Inside `stdlib/compiler/` keep the stricter discipline (pre-bind every call operand, including in `str-concat` nests): there a miscompile breaks the fixed point.

## See Also

- [boot-bind-calls-before-binop](boot-bind-calls-before-binop.md)
