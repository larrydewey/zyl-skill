# gen-per-type-instances

> Write generic code freely: a function whose body prints, compares, does arithmetic on, or calls a trait method on a type parameter is compiled once per concrete argument-type tuple, so each instance behaves correctly for its types.

## Why It Matters

Every value is one word, so most generic functions share one body. But `(defn show (x) (print x))` cannot pick a print format in a shared body. The type annotation pass marks such functions *trait-generic* and, at each call with concrete argument types, makes an instance typed with those types (spec §6.4). Calls inside an instance can create more instances.

```lisp
(defn same (a b) (= a b))
(defn big (a b) (if (> a b) a b))
(defn show-it (x) (print x))

(defn main ()
  (begin
    (print (same "ab" (str-concat "a" "b")))  ; 1   (same~String,String)
    (print (big "x" "y"))                     ; y   (byte order)
    (print (big 2.5 1.0))                     ; 2.500000
    (show-it 42)                              ; 42
    (show-it (Some "s"))                      ; Some(s)
    0))
```

## Notes

- Instances are named `<key>~T1,T2` (`ZYL_DEBUG_TYPES=1` lists them); at most 32 per function, after which calls use the shared body (word semantics).
- A type variable that only the call site mentions and nothing can fix (the `E` of `(Ok "yes")`) defaults to `Int`.
- A call whose argument types are themselves unknown (conflicting data) stays on the shared body.
- Functions that only move values around (`first-of`, `list-length`, `vec-push`) are not instantiated: they don't need to be.

## See Also

- [gen-unannotated-is-polymorphic](gen-unannotated-is-polymorphic.md)
- [trait-static-dispatch](trait-static-dispatch.md)
- [gen-monomorphization-naming](gen-monomorphization-naming.md)
