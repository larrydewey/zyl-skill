# fn-types-drive-codegen

> Let inference type your values: `print`, `=`/`<` and arithmetic follow the inferred type of any expression (fields, pattern binders, captures, container elements, generic results); annotate only to document intent, and watch for data that has no single type.

## Why It Matters

Code generation picks `%lld`/`%f`/`%s`, String comparison (`zyl_cstr_eq`, `zyl_cstr_cmp`) and SSE Float arithmetic from a static kind. That kind now comes from `compiler/type_annotate.zyl`, a Hindley–Milner pass over the whole program, plus literals and annotations. So all of these work without annotations:

```lisp
(deftype Shape (Circle Float) (Rect Float Float))
(defn area (s) (match s (Circle r (* 3.14 (* r r))) (Rect w h (* w h))))
(defn half (x) (/ x 2.0))
(defn shout (s) (print (str-concat s "!")))

(defn main ()
  (let who "bob"
    (let f (fn (n) (str-concat who n))
      (begin
        (print (area (Rect 1.5 2.0)))   ; 3.000000
        (print (half 3.0))              ; 1.500000
        (print (f "!"))                 ; bob!
        (print (= (str-concat "a" "b") "ab"))  ; 1
        0))))
```

## Where it still falls back to a plain word

- **Conflicting types.** A unification failure (a list holding both a `Circle` and a `Rect`, a variable used as both Int and String) marks the involved type variables unknown; codegen then uses only literals and annotations, i.e. the old behavior: a String prints as its address and `=` compares addresses.
- **Unknown FFI results.** An `ffi-call` result is untyped unless the symbol is a known string producer (`zyl_cstr_concat`, `zyl_int_text`, ...). Wrap raw calls in a function whose use pins the type, or pass through a typed helper.
- **The word `-1` sentinels** of `vec-get`/`vec-last` are typed as the element type.

In those cases use `print-string`/`print-float`/`print-int`, `str-eq`, or a `(x Type)` annotation.

## Notes

- Inside a generic body, type-dependent operations are handled by per-type instances: [gen-per-type-instances](gen-per-type-instances.md).
- `ZYL_DEBUG_TYPES=1 zyl file.zyl` prints every function's inferred type; the REPL's `:type expr` shows one expression's.
- Bool still prints `1`/`0`.

## See Also

- [type-inference-does-not-reject](type-inference-does-not-reject.md)
- [data-field-types](data-field-types.md)
- [cg-kind-of](cg-kind-of.md) - the mechanism
