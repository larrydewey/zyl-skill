# test-assert-equal-semantics

> Assert directly on scalars, strings and records (nested ones too — they compare by content); only records with a `Secret` field or an uninferable type fall back to shallow comparison.

## Why It Matters

`assert-equal` compares:

- Ints, Bools: by value. Floats: by value with tolerance `1e-5` (chosen when either side's inferred type is Float, or, when neither side is a String, contains a float literal).
- Strings: by content — `(assert-equal "ab" (str-concat "a" "b"))` passes.
- Structs/ADTs: lowered to `(assert-true (== l r))`, i.e. deep structural equality via the generated `T.==` ([data-equality-shallow](data-equality-shallow.md)), so `(assert-equal (Cons 1 Nil) (Cons 1 Nil))` and `(assert-equal (Some (Some 1)) (Some (Some 1)))` pass. A type with a `Secret` field, or an operand of unknown type, falls back to shallow tag-plus-words `zyl_variant_eq`.

## Good

```lisp
(defstruct Point x y)
(test "equality"
  (begin
    (assert-equal (+ 1 2) 3)
    (assert-equal (struct-get (make-Point 1 2) "x") 1)
    (assert-equal (make-Point 1 2) (make-Point 1 2))   ; passes
    (assert-equal (my-map inc (Cons 1 (Cons 2 Nil))) (Cons 2 (Cons 3 Nil)))   ; deep: passes
    (assert-equal (list-length (my-map inc (Cons 1 (Cons 2 Nil)))) 2)
    (assert-equal (list-sum (my-map inc (Cons 1 (Cons 2 Nil)))) 5)))
```

## Assertions

| Form | Notes |
|---|---|
| `(assert-equal actual expected)` | as above |
| `(assert-true e)` / `(assert-true e "msg")` | outside a test, fails with `msg` (string literal) or `assert-true failed`; inside a test, `FAIL` |
| `(assert-false e)` / with message | |
| `(assert e "msg")` | outside a test, `PANIC: msg` (string literal) or `assert failed`; inside a test, `FAIL` |
| `(assert-fail e)` | evaluates `e`, **always passes** |

Outside a test, a failed assertion prints `PANIC: assert-equal failed` and exits 1.

## See Also

- [data-equality-shallow](data-equality-shallow.md)
- [test-unimplemented-features](test-unimplemented-features.md)
