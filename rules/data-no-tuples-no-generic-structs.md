# data-no-tuples-no-generic-structs

> Use a struct or a single-variant ADT where you want a tuple; use a generic ADT where you want a generic struct; don't rely on `alias`.

## Why It Matters

`(tuple 1 2)` is an undefined function. Generic structs are not supported. `(alias UserId Int)` is accepted but has **no effect** and does not even introduce `UserId` as a name (an annotation `(id UserId)` is accepted only because unknown annotation names are type variables).

## Good

```lisp
(deftype Pair (MkPair A B))
(defn pair-sum (p) (match p (MkPair a b (+ a b))))

(deftype Cell (Cell T))           ; generic "struct" as a one-variant ADT
(defn cell-get (c) (match c (Cell v v)))
```

## Notes

- To return several values, return a struct/ADT and destructure it with `match`.
- Choosing: fixed fields always present → `defstruct`; one of several shapes → `deftype`; small Int→Int table → `collections/map`; recursive → `deftype`. If you write `(if (field? x) ...)` chains, you want an ADT.

## See Also

- [data-adt-declaration](data-adt-declaration.md)
- [type-annotations-guide-codegen](type-annotations-guide-codegen.md)
