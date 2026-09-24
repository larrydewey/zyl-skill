# gen-unannotated-is-polymorphic

> Write generic functions by leaving parameters unannotated; top-level functions get polymorphic types (let-polymorphism) and each call instantiates them.

## Why It Matters

The type annotation pass infers top-level functions one strongly connected component of the call graph at a time and generalizes them, so each call site instantiates a fresh copy of the type: one function serves `Int` and `String` in the same program, and its result has the instantiated type. Bodies are shared unless they depend on the type ([gen-per-type-instances](gen-per-type-instances.md)).

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

(defn main ()
  (begin
    (print (count-items (Cons 1 (Cons 2 Nil))))   ; 2
    (print (count-items (Cons "a" Nil)))          ; 1
    (print (first-of "a" 2))                      ; a
    0))
```

## Caveats

- Local `let`s are not generalized.
- Usage constrains nothing: a failed unification is not an error ([type-inference-does-not-reject](type-inference-does-not-reject.md)).

## See Also

- [gen-generic-adts](gen-generic-adts.md)
- [gen-monomorphization-naming](gen-monomorphization-naming.md)
