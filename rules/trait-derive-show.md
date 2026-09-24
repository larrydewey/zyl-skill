# trait-derive-show

> Use `(derive T Show)` (or `(derive T [Show])`) to make `print` show a record; don't expect the other derivable traits to generate anything.

## Why It Matters

`compiler/derive.zyl` expands `(derive T Show)` into an `(impl Show T ...)` whose `show` renders each variant as `Name(field, ...)` and a struct as `Name { f: v, ... }`, calling `Show.show` on each field (instantiated per field type). `print` of any value whose type has a `Show` impl prints its text. Everything else about derive is still a no-op: `Eq`, `Ord`, `Debug`, `Clone`, `Hash` generate nothing, unknown trait names are accepted, `E_TRAIT_NOT_DERIVABLE` is never raised, and `(defstruct+ ... (:derive ...))` is not parsed.

## Good

```lisp
(deftype Shape (Circle Float) (Rect Int Int) (Empty))
(derive Shape Show)
(defstruct Person (name String) (age Int))
(derive Person Show)

(defn main ()
  (begin
    (print (Rect 2 3))                 ; Rect(2, 3)
    (print Empty)                      ; Empty
    (print (make-Person "Ann" 30))     ; Person { name: Ann, age: 30 }
    (print (Show.show (Circle 1.5)))   ; Circle(1.500000)
    0))
```

## Notes

- Derive once per type: two `derive`s naming `Show` define the impl twice (assembler error).
- Works for generic and recursive ADTs: `(StMk "k" (Some 2))` shows `StMk(k, Some(2))`.
- Strings inside shown data are not quoted.
- `==` on records is already deep structural without `Eq`, and `<` compares field words without `Ord` ([data-equality-shallow](data-equality-shallow.md)).
- The REPL keeps a `derive` entry as a definition.

## See Also

- [trait-static-dispatch](trait-static-dispatch.md)
- [fn-print-semantics](fn-print-semantics.md)
