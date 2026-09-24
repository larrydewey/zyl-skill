# match-no-nested-patterns

> Match one constructor level at a time; a field position holds only a name or `_`.

## Why It Matters

A nested constructor pattern such as `(Some (Cons x _) body)` is **accepted without a diagnostic**, but only the outer tag is tested. The inner constructor is never checked, the arm matches whatever the field holds, and inner names bind to garbage when the shape differs. The first nested arm swallows every value with the outer tag.

## Bad

```lisp
(defn first-of-some (o)
  (match o
    (Some Nil -1)          ; matches EVERY Some
    (Some (Cons x _) x)
    (None 0)))
(first-of-some (Some (Cons 7 Nil)))   ; -1, not 7

(deftype L (N) (C Int L))
(defn f (xs) (match xs (C a (C b _) (+ a b)) (_ 77)))
(f (C 5 (N)))                          ; 0, not 77
```

## Good

```lisp
(defn sum-first-two (lst)
  (match lst
    (Cons x rest
      (match rest
        (Cons y _ (+ x y))
        (Nil x)))
    (Nil 0)))

(deftype Expr (Num Int) (Add Expr Expr))
(defn fold-add (e)
  (match e
    (Add a b
      (match a
        (Num x (match b (Num y (Num (+ x y))) (_ e)))
        (_ e)))
    (_ e)))
```

## Notes

- Deep inner matching is also where residual codegen bugs have historically lived; a small helper function per level is often clearer.

## See Also

- [match-arm-shape](match-arm-shape.md)
- [match-guards-literal-arms-only](match-guards-literal-arms-only.md) - no guards on constructor arms either
