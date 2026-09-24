# test-assert-equal-semantics

> Assert on scalars, strings and flat records; for nested data assert on individual fields or a computed count/sum.

## Why It Matters

`assert-equal` compares:

- Ints, Bools: by value. Floats: by value with tolerance `1e-5` (approximate when either side contains a float literal).
- Strings: by content — `(assert-equal "ab" (str-concat "a" "b"))` passes.
- Structs/ADTs: **shallow** — tag plus each field as a raw word. Nested structs/lists/ADTs compare by address, so `(assert-equal (Cons 1 Nil) (Cons 1 Nil))` and `(assert-equal (Some (Some 1)) (Some (Some 1)))` **fail**.

## Good

```lisp
(defstruct Point x y)
(test "equality"
  (begin
    (assert-equal (+ 1 2) 3)
    (assert-equal (struct-get (make-Point 1 2) "x") 1)
    (assert-equal (make-Point 1 2) (make-Point 1 2))   ; flat: passes
    (assert-equal (list-length (my-map inc (Cons 1 (Cons 2 Nil)))) 2)
    (assert-equal (list-sum (my-map inc (Cons 1 (Cons 2 Nil)))) 5)))
```

## Assertions

| Form | Notes |
|---|---|
| `(assert-equal actual expected)` | as above |
| `(assert-true e)` / `(assert-true e "msg")` | message accepted, not printed |
| `(assert-false e)` / with message | |
| `(assert e "msg")` | **does nothing** |
| `(assert-fail e)` | evaluates `e`, **always passes** |

Outside a test, a failed assertion prints `PANIC: assert-equal failed` and exits 1.

## See Also

- [data-equality-shallow](data-equality-shallow.md)
- [test-unimplemented-features](test-unimplemented-features.md)
