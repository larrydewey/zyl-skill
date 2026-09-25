# match-no-nested-patterns

> Match one constructor level at a time: a field position holds only a name or `_`; a nested pattern is `E_NESTED_PATTERN`.

## Why It Matters

Patterns are flat (spec §4.9). A nested constructor pattern such as `(Some (Cons x _) body)` used to be accepted and match on the outer tag alone; it is now rejected, located at the offending field, with the help "bind the field to a name and match it inside the arm body". A prelude constructor in a binder position, `(Some Nil ...)`, is rejected the same way, because it would otherwise bind a variable named `Nil`.

## Bad

```lisp
(defn first-of-some (o)
  (match o
    (Some Nil -1)          ; E_NESTED_PATTERN
    (Some (Cons x _) x)    ; E_NESTED_PATTERN
    (None 0)))

(deftype L (N) (C Int L))
(defn f (xs) (match xs (C a (C b _) (+ a b)) (_ 77)))   ; E_NESTED_PATTERN

(deftype T (Node Int T T) (Leaf))
(defn g (t) (match t (Node v Leaf _ v) (Leaf 0)))
;; NOT caught: your own nullary constructor as a binder is just a variable
;; named Leaf; the arm matches every Node
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

- The check knows the prelude constructors (`Some None Ok Err Cons Nil`) and any `::`-qualified name; a program's own nullary constructor written bare in a binder position is taken as a binder name. Never reuse a constructor name as a binder.
- A guard on a constructor arm, `(Some x (when (> x 0)) x)`, is also `E_NESTED_PATTERN` ([match-guards-literal-arms-only](match-guards-literal-arms-only.md)).
- A small helper function per level is often clearer than deep inner matches.

## See Also

- [match-arm-shape](match-arm-shape.md)
- [match-guards-literal-arms-only](match-guards-literal-arms-only.md)
