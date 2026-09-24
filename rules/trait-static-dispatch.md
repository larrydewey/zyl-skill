# trait-static-dispatch

> Implement traits for any type — structs, multi-variant ADTs, primitives, generic types — and call them on values whose type inference can determine; avoid trait calls on heterogeneous data except over structs.

## Why It Matters

`(Trait.method recv ...)` is resolved from the receiver's inferred type (spec §5.4) and redirected to `Trait.method_Type`. Inside a generic function the call is resolved per instance ([gen-per-type-instances](gen-per-type-instances.md)). Only when the receiver's type is unknown (conflicting data, e.g. one list holding a `Circle` and a `Rect`) does ICNF lowering fall back to a `match` on the receiver's runtime tag, with arms named after the impl types. That fallback is exact for structs (program-unique tags) but an arm named after a multi-variant ADT or a primitive is a catch-all there. A receiver of **known** type with no impl never reaches the fallback: it is a located `E_TRAIT_NOT_FOUND` (`no impl of `Area.area` for type `Int``).

## Good

```lisp
(deftype Shape (Circ Int) (Sq Int))
(deftype Tri (Tri Int))
(trait Area (area (self) Int))
(impl Area Shape (defn area (self) (match self (Circ r (* 3 (* r r))) (Sq s (* s s)))))
(impl Area Tri (defn area (self) (match self (Tri b b))))
(impl Area Int (defn area (self) (* self 100)))

(Area.area (Tri 9))   ; 9
(Area.area 2)         ; 200

(trait Desc (desc (self) String))
(impl Desc Int (defn desc (self) "int"))
(impl Desc Vec (defn desc (self) (str-concat "vec of " (Desc.desc (vec-get self 0)))))
(Desc.desc (vec-push (vec-create-default 1) 7))   ; vec of int
```

## Notes

- Declare the trait with typed signatures, `(trait T (m (self) String))`: the return type types every call.
- An impl for a generic type names the bare type (`(impl Show Vec ...)`).
- A heterogeneous list of structs still dispatches correctly through the fallback; wrap mixed ADTs/primitives in one ADT instead.

## See Also

- [trait-qualified-calls](trait-qualified-calls.md)
- [trait-no-dyn-use-adt-wrapper](trait-no-dyn-use-adt-wrapper.md)
