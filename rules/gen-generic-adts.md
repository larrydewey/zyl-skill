# gen-generic-adts

> Make data generic with ADTs whose field types are unknown uppercase names; don't expect the same-type constraint or generic structs.

## Why It Matters

Generic ADTs are the supported, tested form of generics (`tests/regression/generics.zyl`, `generics-multi-type.zyl`). Type arguments are inferred from field values and never written. One ADT can be used at several types in one program without interference. But `(Make 1 "hi")` for `(deftype Pair (Make T T))` compiles (the constraint is not enforced), and `defstruct` cannot be generic.

## Good

```lisp
(deftype Maybe (Just T) (Nothing))
(deftype Outcome (Success T) (Failure E))
(deftype Seq (Item T (Seq T)) (End))

(defn maybe-or (m d)
  (match m (Just x x) (Nothing d)))

(defn main ()
  (begin
    (print (maybe-or (Just 4) 0))                  ; 4
    (print (maybe-or (Just "hi") "none"))          ; hi
    0))
```

## Notes

- All instances share constructors, layout and `match` code; per-instance names (`Option_Int`) exist only inside type inference.
- `derive` works on a generic ADT: `(deftype Box (Bx A))` + `(derive Box Show Eq Ord)` gives `Bx(3)`, `Bx(s)`, and `Eq.eq`/`Ord.compare` per instantiation (Strings by content). Write the type parameter as an uppercase name (`T`, `A`); with a lowercase field type such as `a` the String instance shows an address and `Eq.eq` compares words.
- No higher-kinded types, associated types or const generics: pass operations in as function arguments.

## See Also

- [data-adt-declaration](data-adt-declaration.md)
- [data-no-tuples-no-generic-structs](data-no-tuples-no-generic-structs.md)
