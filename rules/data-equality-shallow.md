# data-equality-shallow

> `==` and `<` on structs/ADTs compare one level deep; compare nested values and strings field by field yourself.

## Why It Matters

`==`/`!=` compare the tag and then each field **word**; `<`, `>`, `<=`, `>=` compare fields lexicographically (runtime `zyl_variant_eq`/`zyl_variant_cmp`, using the hidden size header). A field holding a string, list or other ADT is compared by **address**. Two separately built `(Cons 1 (Cons 2 Nil))` are not equal; nor are two `(Some "a")` built from different string allocations.

## Bad

```lisp
(== (Cons 1 (Cons 2 Nil)) (Cons 1 (Cons 2 Nil)))   ; 0: tails differ by address
(assert-equal (Some (Some 1)) (Some (Some 1)))     ; FAIL
```

## Good

```lisp
(== (Some 1) (Some 1))                  ; 1: one level, Int fields
(defstruct Pt (x) (y))
(== (make-Pt 1 2) (make-Pt 1 2))        ; 1
(< (make-Pt 1 2) (make-Pt 2 0))         ; 1

(defn list-eq (a b)                     ; deep equality: write it
  (match a
    (Nil (match b (Nil 1) (Cons _ _ 0)))
    (Cons x xs (match b
                 (Nil 0)
                 (Cons y ys (if (== x y) (list-eq xs ys) 0))))))
```

## Notes

- Different struct types are never equal (tag differs), but mixing them raises no type error.
- `derive Eq/Ord` is a no-op: you get exactly this behavior with or without it.
- Appendix C of the book says structs compare "by identity"; the chapters (2, 15, 18, 20) and the runtime describe the shallow structural comparison above. When correctness matters, compare fields explicitly.
- `print` of a struct/ADT prints its address (compiled); the REPL prints structurally.

## See Also

- [trait-derive-noop](trait-derive-noop.md)
- [test-assert-equal-semantics](test-assert-equal-semantics.md)
