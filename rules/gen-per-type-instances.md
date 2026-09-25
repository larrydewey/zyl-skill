# gen-per-type-instances

> Write generic code freely: a function whose body prints, compares, does arithmetic on, or calls a trait method on a type parameter is compiled once per concrete argument-type tuple, at every call and every use as a value; nothing is dispatched at run time.

## Why It Matters

Every value is one word, so most generic functions share one body. But `(defn show (x) (print x))` cannot pick a print format in a shared body. The type pass marks such functions *trait-generic* and, at each call with concrete argument types, makes an instance typed with those types (spec §6.4). A function passed as a value (`(apply-each show-it xs)`) is specialized the same way, for the type it is used at. Calls inside an instance can create more instances. The generic originals are then dropped: there is no run-time trait dispatch, so every trait call resolves statically, and one whose receiver type stays unknown is `E_CANNOT_INFER`.

```lisp
(defn same (a b) (= a b))
(defn big (a b) (if (> a b) a b))
(defn show-it (x) (print x))
(defn apply-each (f xs)
  (match xs (Nil unit) (Cons x rest (begin (f x) (apply-each f rest)))))

(defn main ()
  (begin
    (print (same "ab" (str-concat "a" "b")))  ; 1   (same~String,String)
    (print (big "x" "y"))                     ; y   (byte order)
    (print (big 2.5 1.0))                     ; 2.500000
    (show-it 42)                              ; 42
    (show-it (Some "s"))                      ; Some(s)
    (apply-each show-it (list "a" "b"))       ; a b (show-it~String as a value)
    0))
```

## Notes

- Instances are named `<key>~T1,T2` (`ZYL_DEBUG_TYPES=1` lists them) ([gen-monomorphization-naming](gen-monomorphization-naming.md)).
- A function that would need more than 256 instances is `E_CANNOT_INFER`; there is no shared-body fallback.
- A type variable that only the call site mentions and nothing can fix (the `E` of `(Ok "yes")`, the element type of `Nil`) is defaulted: `(show-it (Ok "yes"))` prints `Ok(yes)`, `(show-it Nil)` prints `[]`.
- Functions that only move values around (`list-length`, `vec-push`, `option-map`) are not instantiated: they do not need to be.

## See Also

- [gen-unannotated-is-polymorphic](gen-unannotated-is-polymorphic.md)
- [trait-static-dispatch](trait-static-dispatch.md)
- [gen-monomorphization-naming](gen-monomorphization-naming.md)
