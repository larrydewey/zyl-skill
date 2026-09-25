# trait-no-dyn-use-adt-wrapper

> Model open "trait object" designs with an ADT wrapper, a list of structs, or function values; there is no `dyn`.

## Why It Matters

No trait objects, vtables, default methods, supertraits or associated types, and no run-time dispatch: every trait call resolves to one impl at compile time ([trait-static-dispatch](trait-static-dispatch.md)). A list holds one element type, so `[(make-Circle 1) (make-Rect 1 2)]` is `E_TYPE_MISMATCH`, and a trait is not a type: a parameter annotated with a trait name is `E_MALFORMED_PARAMETER`. Choose one of the supported shapes.

## Good

```lisp
(defstruct Circle (r))
(defstruct Rect (w) (h))
(deftype Shape (CircleShape Circle) (RectShape Rect))   ; closed set, exhaustive

(defn shape-area (s)
  (match s
    (CircleShape c (* 3 (* (struct-get c "r") (struct-get c "r"))))
    (RectShape r (* (struct-get r "w") (struct-get r "h")))))

;; or pass behavior as a function value (list-fold is in collections/collections)
(defn total (area-fn xs) (list-fold (fn (acc x) (+ acc (area-fn x))) 0 xs))
(total shape-area [(CircleShape (make-Circle 1)) (RectShape (make-Rect 2 3))])   ; 9
```

## Notes

- Instead of an associated type, return a concrete type: `(trait Iterator (next (self) (Option Int)))`, implemented per type. No collection implements an `Iterator`; `for` is a condition loop.

## See Also

- [trait-static-dispatch](trait-static-dispatch.md)
