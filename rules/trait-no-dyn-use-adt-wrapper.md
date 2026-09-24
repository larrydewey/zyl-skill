# trait-no-dyn-use-adt-wrapper

> Model open "trait object" designs with an ADT wrapper, a list of structs, or function values; there is no `dyn`.

## Why It Matters

No trait objects, vtables, default methods, supertraits or associated types. Capability types cannot be impl targets. Choose one of the supported shapes.

## Good

```lisp
(defstruct Circle (r))
(defstruct Rect (w) (h))
(deftype Shape (CircleShape Circle) (RectShape Rect))   ; closed set, exhaustive

(defn shape-area (s)
  (match s
    (CircleShape c (* 3 (* (struct-get c "r") (struct-get c "r"))))
    (RectShape r (* (struct-get r "w") (struct-get r "h")))))

;; or pass behavior as a function value
(defn total (area-fn xs) (list-fold (fn (acc x) (+ acc (area-fn x))) 0 xs))
```

## Notes

- Instead of an associated type, return a concrete ADT: `(trait Iterator (next (self T) (Option T)))`. No collection implements `Iterator`; `for` is a condition loop.

## See Also

- [trait-static-dispatch](trait-static-dispatch.md)
