# data-equality-shallow

> `==`/`!=` on structs/ADTs compare deeply by content; `<`/`>` still compare raw field words, so order nested or String fields yourself.

## Why It Matters

The type pass generates a per-type equality function `T.==` for each struct/ADT (specialized per element type for generic ones): it matches both operands and compares field pairs with `==`, so nested ADTs, lists (element-wise), Strings (content) and Floats compare by value, wherever the values come from (literals, variables, parameters, call results). `=` is `==`. Ordering is different: `<`, `>`, `<=`, `>=` call runtime `zyl_variant_cmp`, lexicographic over the raw field **words** (hidden size header), so String, list and nested-ADT fields order by **address**.

## Bad

```lisp
(< (Some "b") (Some "a"))                 ; by address: 1 here, meaningless
(defstruct Tok (k Int) (text Secret))
(== t1 t2)                                ; Secret field: shallow word compare
```

## Good

```lisp
(== (Cons 1 (Cons 2 Nil)) (Cons 1 (Cons 2 Nil)))   ; 1
(== (Some "a") (Some (str-concat "" "a")))         ; 1: String content
(assert-equal (Some (Some 1)) (Some (Some 1)))     ; passes
(defstruct Pt (x) (y))
(< (make-Pt 1 2) (make-Pt 2 0))                    ; 1: Int fields order fine

(defn name-lt (a b)                                ; order by String content: write it
  (match a (Some x (match b (Some y (< x y)) (None 0))) (None 1)))
```

## Notes

- Types with a `Secret` field get no `T.==`; they keep the shallow tag-plus-words `zyl_variant_eq`. So does any operand whose type inference cannot determine (a conflict makes it unknown).
- Different struct types are never equal (tag differs), but mixing them raises no type error.
- `derive Eq/Ord` is a no-op and not needed (only `derive Show` generates code).
- Appendix C of the book says structs compare "by identity"; that is wrong.
- `print` of a struct/ADT prints its address (compiled) unless it has a `Show` impl; the REPL prints structurally.

## See Also

- [trait-derive-show](trait-derive-show.md)
- [test-assert-equal-semantics](test-assert-equal-semantics.md)
- [det-no-address-dependent-output](det-no-address-dependent-output.md)
