# data-adt-declaration

> Declare sum types with `(deftype Name (Variant FieldType...) ...)`; an unknown uppercase field type is a type parameter.

## Why It Matters

This is the core modelling tool: every "one of several shapes" value, every result, every tree. Field types are type *names* used for inference and monomorphization, not checked constraints: `(Circle "x")` compiles even if declared `(Circle Int)`.

## Good

```lisp
(deftype Light (Red) (Yellow) (Green))          ; enum-like; (Red) or bare Red
(deftype Shape (Circle Int) (Rect Int Int))     ; payloads
(deftype Maybe (Just T) (Nothing))              ; generic, one parameter
(deftype Either (Left L) (Right R))             ; two parameters
(deftype Pair (Make T T))                       ; T repeated: ONE parameter
(deftype Tree (Node T (Tree T) (Tree T)) (Leaf)); recursive + generic

(defn tree-sum (t)
  (match t
    (Leaf 0)
    (Node v l r (+ v (+ (tree-sum l) (tree-sum r))))))
```

## Notes

- Construction: `(Circle 5)`, `Red` or `(Red)` for nullary, `(Node 1 (Leaf) (Leaf))`.
- A variant name repeated inside one `deftype` is `E_DUPLICATE_VARIANT`.
- Recursive fields are ordinary pointer words: no `Box` needed or available.
- Tags are 0-based in declaration order per `deftype`. Each variant block is sized to its own fields; nullary variants are still heap blocks holding just the tag.
- The spec's same-type constraint (`(Make 1 "x")` should fail for `(Make T T)`) is not enforced.
- `Option`, `Result`, `List` already exist in the prelude.

## See Also

- [data-unique-variant-names](data-unique-variant-names.md)
- [data-no-redeclare-prelude](data-no-redeclare-prelude.md)
- [gen-generic-adts](gen-generic-adts.md)
