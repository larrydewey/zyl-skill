# match-arm-shape

> Write arms as `(Variant binder... body)` with exactly one binder (a name or `_`) per field; the grouped `((Variant binder...) body)` form means the same.

## Why It Matters

The arm is positional: the last element is the body, everything between the constructor and the body binds fields in declaration order. The number of binders is **not checked**: `(Mk a a)` for a two-field `Mk` binds `a` to the first field and returns it, and extra binders read past the fields. A nullary arm is `(None body)`, not `None body`. A binder must be a plain name: a constructor or a guard there is `E_NESTED_PATTERN` ([match-no-nested-patterns](match-no-nested-patterns.md)).

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
- Every arm must produce the same type: `(match l (Red 1) (Green "g"))` is `E_TYPE_MISMATCH`.
- A binder has its field's declared type, or the instantiated type parameter ([data-field-types](data-field-types.md)).

## See Also

- [match-no-nested-patterns](match-no-nested-patterns.md)
- [match-arm-complex](match-arm-complex.md)
