# data-struct-immutable-rebind

> Struct fields never change: to "update" one, build a new struct and rebind a `let-mut` name to it.

## Why It Matters

`(set! (struct-get p "x") 10)` is rejected by the parser with `E_MUT_CONFLICT`. Capabilities belong to bindings, not fields; there is no per-field mutability and no field-level aliasing. This is a core design decision, not a missing feature.

## Bad

```lisp
(set! (struct-get p "x") 10)
;; E_MUT_CONFLICT: set! target must be a plain variable name bound via let-mut
```

## Good

```lisp
(defstruct Point (x) (y))
(defn with-x (p nx) (make-Point nx (struct-get p "y")))

(defn main ()
  (let-mut p (make-Point 0 0)
    (begin
      (set! p (with-x p 10))
      (print (struct-get p "x"))   ; 10
      0)))
```

## Notes

- In pure code, don't use `let-mut` at all: thread the value through function results (`count-entry st e` returns a new `Stats`).
- Because fields are immutable, aliasing a struct pointer is always safe. Collections (`Vec`/`Map`/`Set`) are the exception: they write into shared buffers.

## See Also

- [data-reconstruct-field-order](data-reconstruct-field-order.md) - the rebuild trap
- [own-let-mut-only-set](own-let-mut-only-set.md)
