# gen-unannotated-is-polymorphic

> Write generic functions by leaving parameters unannotated; each call site is inferred separately and all sites share one compiled body.

## Why It Matters

Every value is one word, so an unannotated function already accepts arguments of any type. Inference does not generalize a scheme; it re-infers the body per call site with that site's argument types, caching under `name::ArgType1,ArgType2`. Recursive calls with the same types reuse the cache. No specialized copies are generated for user functions, so generics add no code size.

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
    (print-string (first-of "a" 2))               ; a  (typed printer!)
    0))
```

## Caveats

- Print polymorphic results with typed printers ([type-polymorphic-results-typed-printers](type-polymorphic-results-typed-printers.md)).
- Operators are not overloaded ([gen-operators-not-overloaded](gen-operators-not-overloaded.md)).
- Usage constrains nothing: failed unification is not an error.

## See Also

- [gen-generic-adts](gen-generic-adts.md)
- [gen-monomorphization-naming](gen-monomorphization-naming.md)
