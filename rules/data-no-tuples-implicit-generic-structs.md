# data-no-tuples-implicit-generic-structs

> Group values with `(tuple a b)` and read them with `(tuple-get t 0)` (a literal index), use a struct or single-variant ADT when the parts deserve names, make a struct generic by leaving fields untyped or typing them with uppercase names, and use `alias` to name a type.

## Why It Matters

`(tuple e...)` builds a value of a generated single-variant ADT whose type includes every element type, so `(tuple 1 2)` and `(tuple "a" "b")` are different types. `(tuple-get t i)` takes an integer literal: a variable index is `E_TYPE_MISMATCH` ("a tuple index is an integer literal"), and an index past the end is a compile-time `E_INDEX_OUT_OF_BOUNDS`. There is no tuple pattern in `match`.

Generic structs exist implicitly: every untyped field and every unknown uppercase field type is a type parameter of the struct, in declaration order, and each construction instantiates it. `(defstruct Cell (v))` gives `(make-Cell "s") : (Cell String)` and `(make-Cell 2) : (Cell Int)` in one program, and an annotation can apply it: `(e (Entry Int))`. `(alias UserId Int)` makes `UserId` another name for `Int` (transparent): `(defn g ((id UserId)) id)` accepts an `Int` and rejects a String with `E_TYPE_MISMATCH`.

## Good

```lisp
(alias UserId Int)
(deftype Pair (MkPair A B))
(defn pair-sum (p) (match p (MkPair a b (+ a b))))

(defstruct Cell (v))                       ; generic in v
(defn cell-get (c) c.v)

(defstruct Entry (key String) (val))       ; generic in val only
(defn lookup ((e (Entry Int))) (+ 1 e.val))
(defn g ((id UserId)) id)

(defn main ()
  (begin
    (print (cell-get (make-Cell "s")))         ; s
    (print (+ 1 (cell-get (make-Cell 2))))     ; 3
    (print (pair-sum (MkPair 1 2)))            ; 3
    (print (lookup (make-Entry "k" 4)))        ; 5
    (print (g 5))                              ; 5
    (let t (tuple 1 "a") (print (tuple-get t 1)))  ; a
    0))
```

## Notes

- To return several values, return a tuple, or a struct/ADT and destructure it with `match` or field reads.
- An accidental untyped field makes a struct generic without your noticing; type fields you mean to be concrete ([data-field-types](data-field-types.md)).
- Choosing: fixed fields always present, `defstruct`; one of several shapes, `deftype`; small Int-to-Int table, `collections/intmap`; recursive, `deftype`. If you write `(if (field? x) ...)` chains, you want an ADT.

## See Also

- [data-adt-declaration](data-adt-declaration.md)
- [gen-generic-adts](gen-generic-adts.md)
- [type-annotations-constrain](type-annotations-constrain.md)
