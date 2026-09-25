# gen-generic-adts

> Make data generic with ADTs (or structs) whose field types are unknown **uppercase** names; each name is one type parameter, checked at every construction and use.

## Why It Matters

Generic ADTs are the main form of generics (`tests/regression/generics.zyl`, `generics-multi-type.zyl`). Type arguments are inferred from field values and never written at construction; in an annotation you apply the type: `(m (Maybe Int))`. One ADT can be used at several types in one program without interference. The checker enforces the parameters: `(Make 1 "hi")` for `(deftype Pair (Make T T))` is `E_TYPE_MISMATCH`, and a `(Just "x")` payload cannot be used as an Int.

## Good

```lisp
(deftype Maybe (Just T) (Nothing))
(deftype Outcome (Success T) (Failure E))
(deftype Seq (Item T (Seq T)) (End))

(defn maybe-or (m d)
  (match m (Just x x) (Nothing d)))
(defn only-ints ((m (Maybe Int))) (maybe-or m 0))

(defn main ()
  (begin
    (print (maybe-or (Just 4) 0))                  ; 4
    (print (maybe-or (Just "hi") "none"))          ; hi
    (print (only-ints (Just 2)))                   ; 2
    0))
```

## Bad

```lisp
(deftype Box (Bx a))                       ; lowercase: NOT a type parameter
(+ 1 (match (Bx "s") (Bx v v)))            ; compiles and adds the string's address
```

A lowercase unknown field type is a fresh type unrelated to the ADT's parameters, so what you read back is never checked (a soundness hole). Always write `T`, `A`, `K`, `V`.

## Notes

- All instances share constructors, layout and `match` code. Only functions whose body depends on the type (printing, comparing, arithmetic, a trait call) are specialized ([gen-per-type-instances](gen-per-type-instances.md)), and `==` on a generic ADT uses a `T.==` specialized per element type.
- `derive` works on a generic ADT: `(deftype Box (Bx A))` + `(derive Box Show Eq Ord)` gives `Bx(3)`, `Bx(s)`, and `Eq.eq`/`Ord.compare` per instantiation (Strings by content).
- Structs are generic too: each untyped field, and each unknown uppercase field type, is a parameter of the struct ([data-no-tuples-implicit-generic-structs](data-no-tuples-implicit-generic-structs.md)).
- No higher-kinded types, associated types, trait bounds or const generics: pass operations in as function arguments.

## See Also

- [data-adt-declaration](data-adt-declaration.md)
- [data-no-tuples-implicit-generic-structs](data-no-tuples-implicit-generic-structs.md)
