# match-arm-complex

> In a match arm, never combine a constant with two or more calls in one arithmetic expression; bind the calls with `let` first.

## Why It Matters

An arm body shaped like `(+ 1 (f n m) (g n))` is rejected with a located `E_MATCH_ARM_COMPLEX` ("match arm combines a constant with multiple calls", help: bind the calls with `let` first, or nest the sum through a helper function). The error is a guard against a shape older stage binaries miscompiled (it computed 0). It is raised during ICNF lowering, so `zyl check`, which stops after type inference, does not report it; a build does. The broader old rule ("any binop over two calls computes 0") no longer reproduces: `(+ (size l) (size r))` and `(+ 1 (+ (size l) (size r)))` compile correctly.

## Bad

```lisp
(deftype Tree (Node Int Tree Tree) (Leaf))
(defn size (t)
  (match t
    (Leaf 0)
    (Node _ l r (+ 1 (size l) (size r)))))   ; E_MATCH_ARM_COMPLEX, located
```

## Good

```lisp
(deftype Tree (Node Int Tree Tree) (Leaf))
(defn size (t)
  (match t
    (Leaf 0)
    (Node _ l r
      (let a (size l)
        (let b (size r)
          (+ 1 a b))))))

(defn size2 (t)                                ; or nest the sum explicitly
  (match t (Leaf 0) (Node _ l r (+ 1 (+ (size2 l) (size2 r))))))
```

## Notes

- N-ary arithmetic folds left: `(+ a b c)` is `(+ (+ a b) c)`.
- Inside `stdlib/compiler/` keep the stricter discipline (pre-bind every call operand, including in `str-concat` nests): there a miscompile breaks the fixed point.

## See Also

- [boot-match-arm-call-sums](boot-match-arm-call-sums.md)
