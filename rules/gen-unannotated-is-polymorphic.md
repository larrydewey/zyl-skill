# gen-unannotated-is-polymorphic

> Write generic functions by leaving parameters unannotated; top-level functions get polymorphic types (let-polymorphism) and each call instantiates them.

## Why It Matters

The type pass infers top-level functions one strongly connected component of the call graph at a time and generalizes them, so each call site instantiates a fresh copy of the type: one function serves `Int` and `String` in the same program, and its result has the instantiated type. Bodies are shared unless they depend on the type ([gen-per-type-instances](gen-per-type-instances.md)). Within one call, usage still constrains: the instantiated type must be consistent, or it is `E_TYPE_MISMATCH` ([type-sound-checking](type-sound-checking.md)).

## Good

```lisp
(defn my-map (f xs)
  (match xs
    (Nil Nil)
    (Cons x rest (Cons (f x) (my-map f rest)))))

(defn count-items (xs)
  (match xs
    (Nil 0)
    (Cons _ rest (+ 1 (count-items rest)))))

(defn first-of (a _) a)

(defn main ()
  (begin
    (print (count-items (list 1 2)))            ; 2
    (print (count-items (list "a")))            ; 1
    (print (my-map (fn (x) (* x 2)) [1 2 3]))   ; [2, 4, 6]
    (print (first-of "a" 2))                    ; a
    0))
```

## Caveats

- Local `let` bindings are not generalized: `(let id (fn (x) x) ...)` used at Int and String is `E_TYPE_MISMATCH`. Make it a top-level `defn`.
- Top-level `def` values are not generalized either (value restriction), so one `def` vector cannot hold two element types.
- Annotating a parameter with an uppercase type variable (`(a T)`) keeps the function polymorphic while tying parameters together.

## See Also

- [gen-generic-adts](gen-generic-adts.md)
- [gen-monomorphization-naming](gen-monomorphization-naming.md)
