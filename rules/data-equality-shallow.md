# data-equality-shallow

> `==`/`!=` on structs/ADTs compare deeply by content; `<`/`>` still compare raw field words, so order nested or String fields with a derived `Ord.compare` or by hand.

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

(deftype Tok (Num Int) (Word String))
(derive Tok Ord Eq)
(Ord.compare (Word "a") (Word "b"))                ; -1: by String content
(Ord.compare (Num 9) (Word "a"))                   ; -1: variant declaration order first

(defn name-lt (a b)                                ; or write the order by hand
  (match a (Some x (match b (Some y (< x y)) (None 0))) (None 1)))
```

## Notes

- Types with a `Secret` field get no `T.==`; they keep the shallow tag-plus-words `zyl_variant_eq`. So does any operand whose type inference cannot determine (a conflict makes it unknown).
- Different struct types are never equal (tag differs), but mixing them raises no type error.
- `derive Eq` generates `Eq.eq` (structural, the same answer as `==`); `derive Ord` generates `Ord.compare` returning `-1`/`0`/`1` — variant declaration order, then fields lexicographically, Strings by content. `<`/`>` do **not** use it: call `Ord.compare` explicitly. A record with a `Secret` field cannot derive `Eq`/`Ord`/`Hash` (`E_TRAIT_NOT_DERIVABLE`) ([trait-derive-show](trait-derive-show.md)).
- Appendix C of the book says structs compare "by identity"; that is wrong.
- `print` of a struct/ADT prints its address (compiled) unless it has a `Show` impl; the REPL prints structurally.

## See Also

- [trait-derive-show](trait-derive-show.md)
- [test-assert-equal-semantics](test-assert-equal-semantics.md)
- [det-no-address-dependent-output](det-no-address-dependent-output.md)
