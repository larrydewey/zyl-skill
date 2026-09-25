# trait-static-dispatch

> Implement traits for any type — structs, multi-variant ADTs, primitives, generic types — and call them only where the receiver's type is known at compile time: every trait call resolves statically, and there is no run-time dispatch.

## Why It Matters

`(Trait.method recv ...)` is resolved from the receiver's inferred type (spec §5.4) and redirected to the lifted impl `Trait.method_Type`. A function that calls a trait method on a parameter is trait-generic: it is specialized per type at every call *and every use as a value*, and the generic original is dropped ([gen-per-type-instances](gen-per-type-instances.md)); more than 256 instances of one function is `E_CANNOT_INFER`. The old run-time tag dispatch (`ic-trait-dispatch`) is gone, so:

- a receiver whose type is known but has no impl is a located `E_TRAIT_NOT_FOUND` ("no impl of `Area.area` for type `String`", `= help: add (impl Area String ...)`);
- a receiver whose type nothing determines (for example the untyped result of `(receive)`) is `E_CANNOT_INFER`, never defaulted;
- a heterogeneous collection cannot be built at all: `[(make-Circle 1) (make-Rect 1 2)]` is `E_TYPE_MISMATCH` (one list, one element type). Wrap the variants in one ADT and implement the trait for it ([trait-no-dyn-use-adt-wrapper](trait-no-dyn-use-adt-wrapper.md)).

## Good

```lisp
(use collections/vec)

(deftype Shape (Circ Int) (Sq Int))
(deftype Tri (Tri Int))
(trait Area (area (self) Int))
(impl Area Shape (defn area (self) (match self (Circ r (* 3 (* r r))) (Sq s (* s s)))))
(impl Area Tri (defn area (self) (match self (Tri b b))))
(impl Area Int (defn area (self) (* self 100)))

(defn total-area (xs)                        ; trait-generic: one instance per element type
  (match xs (Nil 0) (Cons h t (+ (Area.area h) (total-area t)))))

(trait Desc (desc (self) String))
(impl Desc Int (defn desc (self) "int"))
(impl Desc Vec (defn desc (self) (str-concat "vec of " (Desc.desc (vec-get self 0)))))

(defn main ()
  (begin
    (print (Area.area (Tri 9)))                         ; 9
    (print (Area.area 2))                               ; 200
    (print (total-area [(Sq 1) (Circ 1)]))              ; 4
    (print (total-area [1 2]))                          ; 300
    (print (Desc.desc (vec-push (vec-create-default 1) 7)))   ; vec of int
    0))
```

## Bad

```lisp
(Area.area "s")                                   ; E_TRAIT_NOT_FOUND: no impl for String
(Area.area (receive))                             ; E_CANNOT_INFER: the receiver's type is unknown
(total-area [(Tri 1) (Sq 2)])                     ; E_TYPE_MISMATCH: Tri and Shape in one list
```

## Notes

- Declare the trait with typed signatures, `(trait T (m (self) String))`: the return type types every call, and each impl's method is checked against the declaration (a different parameter count is `E_TYPE_MISMATCH`). The parameter list is a list: `(trait Area (area self))` is `E_MALFORMED_FORM`.
- `Self` in a declaration is the receiver's type: `Ord.compare` takes two values of one type, so `(Ord.compare 1 "one")` is `E_TYPE_MISMATCH`.
- An impl for a generic type names the bare type (`(impl Show Vec ...)`).
- A trait-generic function passed as a value is specialized too: `(let f show-all (f 5))` works.

## See Also

- [trait-qualified-calls](trait-qualified-calls.md)
- [trait-no-dyn-use-adt-wrapper](trait-no-dyn-use-adt-wrapper.md)
