# match-arm-complex

> In a match arm, never combine a constant with two or more calls in one arithmetic expression; bind the calls with `let` first.

## Why It Matters

An arm body shaped like `(+ 1 (f n m) (g n))` is rejected during ICNF lowering with `E_MATCH_ARM_COMPLEX: match arm combines a constant with multiple calls - nest sums through helper functions`. The error is a guard against a shape older stage binaries miscompiled (it computed 0), and it carries **no source location**, so search your `match` arms for the shape. The broader old rule ("any binop over two calls computes 0") no longer reproduces: `(+ (size l) (size r))` and `(+ 1 (+ (size l) (size r)))` compile correctly.

## Bad

```lisp
(match t
  (Node v l r (+ 1 (size l) (size r))))   ; E_MATCH_ARM_COMPLEX (unlocated)
```

## Good

```lisp
(match t
  (Node _ l r
    (let a (size l)
      (let b (size r)
        (+ 1 a b)))))

;; or nest the sum explicitly
(match t (Node _ l r (+ 1 (+ (size l) (size r)))))
```

## Notes

- N-ary arithmetic folds left: `(+ a b c)` is `(+ (+ a b) c)`.
- Inside `stdlib/compiler/` keep the stricter discipline (pre-bind every call operand, including in `str-concat` nests): there a miscompile breaks the fixed point.

## See Also

- [boot-match-arm-call-sums](boot-match-arm-call-sums.md)
