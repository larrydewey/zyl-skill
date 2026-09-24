# trait-qualified-calls

> Call trait methods by qualified name, `(Trait.method receiver args...)`, with the receiver first.

## Why It Matters

Each impl method is lifted to a top-level function `Trait.method_Type`, and `(Trait.method ...)` is resolved to one of them from the receiver's inferred type ([trait-static-dispatch](trait-static-dispatch.md)). A bare `(method r)` finds nothing and fails **at link time** with an undefined reference. The `.` is part of the identifier.

## Good

```lisp
(defstruct Rect (w) (h))
(defstruct Circle (r))
(trait Area (area self))
(impl Area Rect
  (defn area (self) (* (struct-get self "w") (struct-get self "h"))))
(impl Area Circle
  (defn area (self) (* 3 (* (struct-get self "r") (struct-get self "r")))))

(defn main ()
  (begin
    (print (Area.area (make-Rect 3 4)))    ; 12
    (print (Area.area (make-Circle 2)))    ; 12
    0))
```

## Notes

- Further parameters follow the receiver: `(defn scale (self k) ...)` called `(Scale.scale r 3)`.
- The method's parameters are whatever the `defn` in the impl says; they are not checked against the `trait` declaration, and a missing method is not reported. The declaration's return type does type every call.
- No default method bodies, no supertraits, no `where` clauses, no associated types.
- Stdlib traits: `Show` (prelude, `core/show`; `show` returns a String) and `OutputStream` in `io/io` (`write`, `flush`) for `Stdout` (`make-stdout`) and `StringBuffer` (`make-string-buffer`).
- A trait call may be written anywhere, including as `struct-get`'s first argument.

## See Also

- [trait-static-dispatch](trait-static-dispatch.md)
- [trait-derive-show](trait-derive-show.md)
