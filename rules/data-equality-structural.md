# data-equality-structural

> `==`/`!=` on structs and ADTs compare structurally by content through a generated `T.==`; `<`/`>` on them are type errors, so order records with a derived `Ord.compare` or by hand.

## Why It Matters

The type pass generates a per-type equality function `T.==` for each struct and ADT (specialized per element type for generic ones): it matches both operands and compares field pairs with `==`, so nested ADTs, lists (element-wise), Strings (content) and Floats compare by value, wherever the values come from. `=` is `==`. Both operands must have one type: comparing two different struct types is `E_TYPE_MISMATCH`. Ordering operators take only Int, Float and String; on a record they are `E_TYPE_MISMATCH: ordering on (Option String)` (codegen used to compare addresses).

## Bad

```lisp
(< (Some "b") (Some "a"))                 ; E_TYPE_MISMATCH: ordering on (Option String)
(defstruct Pt (x Int) (y Int))
(< (make-Pt 1 2) (make-Pt 2 0))           ; E_TYPE_MISMATCH: ordering on Pt
(== (view-of "ab") (view-slice "xab" 1 2)) ; false: compares base, offset, length
```

## Good

```lisp
(== (Cons 1 (Cons 2 Nil)) (Cons 1 (Cons 2 Nil)))   ; true (`print` shows 1)
(== (Some "a") (Some (str-concat "" "a")))         ; true: String content
(assert-equal (Some (Some 1)) (Some (Some 1)))     ; passes

(deftype Tok (Num Int) (Word String))
(derive Tok Ord Eq)
(Ord.compare (Word "a") (Word "b"))                ; -1: by String content
(Ord.compare (Num 9) (Word "a"))                   ; -1: variant declaration order first
(Ord.compare (Some 2) (Some 1))                    ; 1: the prelude types implement Ord

(defn name-lt (a b)                                ; or write the order by hand
  (match a (Some x (match b (Some y (< x y)) (None false))) (None true)))
```

## Notes

- `T.==` compares the representation. A type whose meaning differs from its fields needs its own equality: views compare by bytes only through `view-eq`/`Eq.eq` ([data-views-and-slices](data-views-and-slices.md)).
- A type with a `Secret` field gets no `T.==`: `==` on it is the shallow tag-plus-words `zyl_variant_eq` (Strings inside by address). It cannot derive `Eq`/`Ord`/`Hash` (`E_TRAIT_NOT_DERIVABLE`).
- `derive Eq` generates `Eq.eq` (the same answer as `==`); `derive Ord` generates `Ord.compare` returning `-1`/`0`/`1`: variant declaration order, then fields lexicographically, Strings by content ([trait-derive-show](trait-derive-show.md)).
- `print` of a struct/ADT without a `Show` impl prints its address in a compiled program (a container such as `Some` prints raw when its payload has no `Show`); the REPL prints structurally.

## See Also

- [trait-derive-show](trait-derive-show.md)
- [test-assert-equal-semantics](test-assert-equal-semantics.md)
- [det-no-address-dependent-output](det-no-address-dependent-output.md)
