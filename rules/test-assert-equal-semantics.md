# test-assert-equal-semantics

> `assert-equal` is typed: both sides must have one type (else `E_TYPE_MISMATCH`), and it compares by that type, by content for strings and records; only records with a `Secret` field fall back to a shallow comparison.

## Why It Matters

`assert-equal` unifies its two sides, so comparing an `Int` with a `String`, or a `Bool` result with `1`, is a compile error, not a failing test. Given one type, it compares:

- Ints, Bools: by value.
- Floats: with tolerance `1e-5` (|a − b| ≤ 1e-5), chosen by the inferred type, so a variable holding a Float gets it too.
- Strings: by content — `(assert-equal "ab" (str-concat "a" "b"))` passes.
- Structs/ADTs: by the type's structural equality, the generated `T.==` resolved like `==` ([data-equality-structural](data-equality-structural.md)), so `(assert-equal (Cons 1 Nil) (Cons 1 Nil))` and `(assert-equal (Some (Some 1)) (Some (Some 1)))` pass. A type with a `Secret` field gets no generated equality and falls back to the shallow tag-plus-words `zyl_variant_eq`: Int fields compare by value, but a String field compares by address, so two records equal in content can FAIL.

## Bad

```lisp
(test "mixed" (assert-equal 1 "1"))                      ; E_TYPE_MISMATCH: cannot unify String with Int
(test "bool-as-int" (assert-equal (str-eq "a" "a") 1))   ; E_TYPE_MISMATCH: str-eq is a Bool
(test "int-as-bool" (assert-true 1))                     ; E_TYPE_MISMATCH: assert-true takes a Bool
```

## Good

```lisp
(use core/list)
(defstruct Point x y)
(defn inc (x) (+ x 1))
(defn my-map (f l) (match l (Nil Nil) (Cons h t (Cons (f h) (my-map f t)))))
(defn third ((x Float)) (/ x 3.0))
(test "equality"
  (begin
    (assert-equal (+ 1 2) 3)
    (assert-equal (struct-get (make-Point 1 2) "x") 1)
    (assert-equal (make-Point 1 2) (make-Point 1 2))                          ; by content
    (assert-equal (my-map inc (Cons 1 (Cons 2 Nil))) (Cons 2 (Cons 3 Nil)))   ; deep
    (assert-equal "ab" (str-concat "a" "b"))
    (assert-equal (third 1.0) 0.333333)                                       ; within 1e-5
    (assert-equal (str-eq "a" "a") true)
    (assert-true (str-eq "a" "a"))))
(run-tests)
```

## Assertions

| Form | Notes |
|---|---|
| `(assert-equal actual expected)` | as above |
| `(assert-true e)` / `(assert-true e "msg")` | `e` must be a `Bool`; outside a test, fails with `msg` (string literal) or `assert-true failed`; inside a test, `FAIL` |
| `(assert-false e)` / with message | `e` must be a `Bool` |
| `(assert e "msg")` | `e` must be a `Bool`; outside a test, `PANIC: msg` (string literal) or `assert failed`; inside a test, `FAIL` |
| `(assert-fail e)` | evaluates `e`, **always passes** |

Outside a test, a failed `assert-equal` prints `PANIC: assert-equal failed` and exits 1.

## See Also

- [data-equality-structural](data-equality-structural.md)
- [test-unimplemented-features](test-unimplemented-features.md)
