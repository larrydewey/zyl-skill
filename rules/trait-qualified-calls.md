# trait-qualified-calls

> Call trait methods with dot syntax, `(r.method args...)` or `((expr).method args...)`, picked by the receiver's type; use the qualified `(Trait.method r args...)` when several traits share the name and the receiver's type is unknown.

## Why It Matters

Each impl method is lifted to a top-level function `Trait.method_Type`, and `(Trait.method ...)` is resolved to one of them from the receiver's inferred type ([trait-static-dispatch](trait-static-dispatch.md)). A dot call `(r.m args)` is rewritten to `(zyl-method "m" r args)` before qualification; the type pass picks the only trait declaring `m`, or, among several, the one with an impl for the receiver's known type, then resolves it exactly like `(Trait.m r args)`. `E_TRAIT_NOT_FOUND` (located) for an undeclared method, a type without the impl, or an ambiguous call on a receiver of unknown type (an unannotated parameter): write the qualified name there. The ambiguous case reads "method `nm` is declared by more than one trait: A1.nm, A2.nm"; a qualified `(A1.nm r)` in the same generic function resolves. A qualified `(Trait.m r)` on a receiver of known type with no impl is the same located `E_TRAIT_NOT_FOUND` (`no impl of `Show.show` for type `P``; `= help: add (impl Show P ...)`), and one on a receiver of unknown type is `E_CANNOT_INFER`: there is no run-time dispatch. A bare `(method r)` is `E_UNBOUND_VARIABLE` ("call to undefined function `nm`"). In `Trait.method` (uppercase first) the `.` is part of the identifier.

## Good

```lisp
(defstruct Rect (w) (h))
(defstruct Circle (r))
(trait Area (area (self) Int))
(impl Area Rect
  (defn area (self) (* (struct-get self "w") (struct-get self "h"))))
(impl Area Circle
  (defn area (self) (* 3 (* (struct-get self "r") (struct-get self "r")))))

(defn main ()
  (begin
    (print (Area.area (make-Rect 3 4)))    ; 12
    (print (Area.area (make-Circle 2)))    ; 12
    (print ((make-Rect 3 4).area))         ; 12: dot syntax on an expression
    0))
```

## Notes

- Further parameters follow the receiver: `(trait Scale (scale (self (k Int)) Int))`, implemented `(defn scale (self k) ...)`, called `(Scale.scale r 3)` or `(r.scale 3)`.
- Each impl method is checked against the `trait` declaration (a different parameter count or type is `E_TYPE_MISMATCH`, reported without a location); a method the impl leaves out is not reported until something calls it. A declaration's parameter list must be a list, `(area (self) Int)`: a bare `(area self)` is `E_MALFORMED_FORM`. The declaration's return type types every call and may be compound, `(trait Dec (dec (self v) (Result Int String)))`, so `print` of the call uses `Show`. Extra `(p Type)` forms between the parameter list and the return type are also parameters: `(dec (self) (v Int) (Result Int String))`. A method declared without a return type takes its impl's return type at resolved calls.
- No default method bodies, no supertraits, no `where` clauses, no associated types.
- Stdlib traits: `Show`, `Debug`, `Eq`, `Ord`, `Hash`, `Clone`, `Secret` (prelude, `core/show`; see [trait-derive-show](trait-derive-show.md)) and `OutputStream` in `io/io` (`write`, `flush`) for `Stdout` (`make-stdout`) and `StringBuffer` (`make-string-buffer`).
- A trait call may be written anywhere, including as `struct-get`'s first argument or before a field: `(Nm.nm p).x`.
- Head position chains fields then a method: `((seg).b.norm 100)` reads field `b` of `(seg)` and calls `norm` on it with `100`.

## See Also

- [trait-static-dispatch](trait-static-dispatch.md)
- [trait-derive-show](trait-derive-show.md)
