# data-no-tuples-implicit-generic-structs

> Use a struct or a single-variant ADT where you want a tuple; make a struct generic by leaving fields untyped or typing them with uppercase names; don't rely on `alias`.

## Why It Matters

`(tuple 1 2)` is `E_UNBOUND_VARIABLE` (no such function). Generic structs do exist, implicitly: every untyped field and every unknown uppercase field type is a type parameter of the struct, in declaration order, and each construction instantiates it. `(defstruct Cell (v))` gives `(make-Cell "s") : (Cell String)` and `(make-Cell 2) : (Cell Int)` in one program, and an annotation can apply it: `(e (Entry Int))`. `(alias UserId Int)` is accepted but has **no effect**: `UserId` in an annotation is just an unknown uppercase name, a type variable, so `(defn g ((id UserId)) id)` accepts a String.

## Good

```lisp
(deftype Pair (MkPair A B))
(defn pair-sum (p) (match p (MkPair a b (+ a b))))

(defstruct Cell (v))                      ; generic in v
(defn cell-get (c) c.v)
(print (cell-get (make-Cell "s")))        ; s
(print (+ 1 (cell-get (make-Cell 2))))    ; 3

(defstruct Entry (key String) (val))      ; generic in val only
(defn lookup ((e (Entry Int))) (+ 1 e.val))
```

## Notes

- To return several values, return a struct/ADT and destructure it with `match` or field reads.
- An accidental untyped field makes a struct generic without your noticing; type fields you mean to be concrete ([data-field-types](data-field-types.md)).
- Choosing: fixed fields always present, `defstruct`; one of several shapes, `deftype`; small Int-to-Int table, `collections/map`; recursive, `deftype`. If you write `(if (field? x) ...)` chains, you want an ADT.

## See Also

- [data-adt-declaration](data-adt-declaration.md)
- [gen-generic-adts](gen-generic-adts.md)
- [type-annotations-constrain](type-annotations-constrain.md)
