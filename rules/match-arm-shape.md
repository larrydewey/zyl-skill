# match-arm-shape

> Write arms as `(Variant binder... body)` with exactly one binder (a name or `_`) per field; the grouped `((Variant binder...) body)` form means the same.

## Why It Matters

The arm is positional: the last element is the body, everything between the constructor and the body binds fields in declaration order. An arm with the wrong number of binders misassigns fields. A nullary arm is `(None body)`, not `None body`.

## Good

```lisp
(match opt
  (Some x (* x 10))
  (None 0))

(match r
  ((Ok v) v)              ; grouped form, identical meaning
  ((Err _) 0))

(defstruct Point (x) (y))
(match p (Point x y (+ x y)))    ; structs match as one-variant ADTs

(match xs
  (Nil Nil)                      ; not `Nil Nil` outside parens
  (Cons x rest (Cons (f x) (my-map f rest))))
```

## Notes

- `match` is an expression; it works in value position (let values, call args, if branches, nested arm bodies).
- The scrutinee is evaluated once.
- Every arm should produce the same type. This is **not checked**; mismatched arm types compile.
- A Float or String bound by a pattern loses its kind for printing/arithmetic ([data-field-kinds](data-field-kinds.md)).

## See Also

- [match-no-nested-patterns](match-no-nested-patterns.md)
- [match-arm-complex](match-arm-complex.md)
