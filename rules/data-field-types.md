# data-field-types

> Declare field types on `deftype` variants and `defstruct` fields: construction is checked against them, a pattern-bound name or field read carries the declared type, and an untyped field makes the record generic in that field.

## Why It Matters

A field is one 8-byte word at run time. Its declared type (recorded at parse time and used by the type pass) is what tells code generation whether that word is a String pointer or a Float bit pattern, and the checker holds every construction to it: `(make-Flag 1)` for `(on Bool)` is `E_TYPE_MISMATCH` (use `true`/`false`). An untyped struct field, `(defstruct P (x) (y))`, is an implicit type parameter of the struct: `(make-P "a" 2)` has type `(P String Int)`, and each use is checked at that type ([data-no-tuples-implicit-generic-structs](data-no-tuples-implicit-generic-structs.md)).

## Good

```lisp
(deftype Shape (Circle Float) (Rect Float Float))
(defstruct Person (name String) (age Int))

(defn area (s)
  (match s
    (Circle r (* 3.14 (* r r)))
    (Rect w h (* w h))))

(defn main ()
  (begin
    (print (area (Rect 1.5 2.0)))                          ; 3.000000
    (print (struct-get (make-Person "Alice" 30) "name"))   ; Alice
    0))
```

## Notes

- Uppercase unknown names in fields are type parameters: `(deftype Pair (Mk A B))`, `(defstruct Cell (v T) (w T))` (both fields one type).
- Lowercase unknown names are not: in a `defstruct` field, `(x a)` is `E_TYPE_MISMATCH: field without a type`; in a `deftype` field, `(Bx a)` is silently unchecked ([gen-generic-adts](gen-generic-adts.md)).
- A field of an applied type works: `(Path (Vec String) Err)`, `(Obj (Map String Val))`.
- Record the fields' types even when inference would find them: it documents the data, puts the error at the construction instead of at a distant use, and keeps the record from becoming generic by accident.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [data-adt-declaration](data-adt-declaration.md)
