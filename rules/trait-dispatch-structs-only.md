# trait-dispatch-structs-only

> Implement multi-impl traits for **structs**; give a trait over an ADT or primitive type a single impl.

## Why It Matters

Dispatch is not static. `trait_dispatch.zyl` rewrites each qualified call into a `match` on the receiver's runtime tag, with one arm per implementing type **named after the type**. Arm names are ordinary constructor patterns:

- A struct is a one-variant ADT named after itself with a program-unique tag → dispatch is correct.
- A trait with exactly one impl → always correct, whatever the type.
- Several impls where one is for a multi-variant ADT or a primitive → **wrong dispatch, silently**. An ADT's name is not one of its variants, so its arm acts as a catch-all; `Int` likewise catches everything.

## Bad

```lisp
(deftype Shape (Circ Int) (Sq Int))
(deftype Tri (Tri Int))
(trait Area (area self))
(impl Area Shape (defn area (self) (match self (Circ r (* 3 (* r r))) (Sq s (* s s)))))
(impl Area Tri (defn area (self) (match self (Tri b b))))
(Area.area (Tri 9))              ; expected 9; runs Shape's impl: 243

;; (impl Show Int ...) next to (impl Show Point ...): a Point runs the Int impl
```

## Good

```lisp
(defstruct Circle (r))
(defstruct Rect (w) (h))
(trait Describe (describe self))
(impl Describe Circle (defn describe (self) (struct-get self "r")))
(impl Describe Rect (defn describe (self) (* (struct-get self "w") (struct-get self "h"))))

;; heterogeneous list of structs: behaves like dynamic dispatch over a closed set
(defn describe-all (xs)
  (match xs
    (Nil 0)
    (Cons x rest (+ (Describe.describe x) (describe-all rest)))))
```

## Notes

- Alternative to traits over ADTs: one function that `match`es the ADT, or wrap each case in a struct.

## See Also

- [trait-qualified-calls](trait-qualified-calls.md)
- [trait-no-dyn-use-adt-wrapper](trait-no-dyn-use-adt-wrapper.md)
