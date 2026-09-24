# data-field-types

> Declare field types on `deftype` variants and `defstruct` fields: a pattern-bound name or `struct-get` result carries the declared type, so Strings and Floats read back from records print and compute correctly.

## Why It Matters

A field is one 8-byte word at run time. Its *declared* type (recorded at parse time and used by the type annotation pass) is what tells code generation whether that word is a String pointer or a Float bit pattern. An untyped struct field `(defstruct P (x) (y))` gets a fresh type variable that usage may or may not pin down.

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

- Uppercase unknown names in fields are type parameters: `(deftype Pair (Mk A B))`.
- A field of an applied type works: `(Path (Vec String) Err)`, `(Obj (Map String Val))`.
- Record the fields' types even when unannotated code would infer them; it documents the data and survives heterogeneous use.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [data-adt-declaration](data-adt-declaration.md)
