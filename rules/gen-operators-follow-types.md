# gen-operators-follow-types

> Use `==`, `<` and arithmetic directly in generic code: each instance uses its own types' operators (Strings compare by content, `<` on Strings is byte order, Floats use SSE); for records, `==` is structural and ordering goes through `Ord.compare` or a comparator argument.

## Why It Matters

A generic body that applies an operator to a type parameter is specialized per argument-type tuple ([gen-per-type-instances](gen-per-type-instances.md)), so `(defn smaller (a b) (if (< a b) a b))` compares numbers for Ints and Floats and text for Strings. The operators constrain what the parameters may be:

| Operator | Allowed operand types |
|---|---|
| `+ - * / %` | two Ints or two Floats (`(+ "a" "b")` is `E_TYPE_MISMATCH: arithmetic on String`) |
| `< > <= >=` | two Ints, two Floats or two Strings (a record is `E_TYPE_MISMATCH: ordering on (Option Int)`) |
| `== = !=` | any one type; on structs and ADTs the generated structural `T.==` ([data-equality-structural](data-equality-structural.md)) |

So `(smaller (Some 1) (Some 2))` is rejected at the call. To order records, take a comparator.

## Good

```lisp
(defn smaller (a b) (if (< a b) a b))
(defn smaller-by (lt a b) (if (lt a b) a b))

(defn main ()
  (begin
    (print (smaller "banana" "apple"))                                       ; apple
    (print (smaller 3 5))                                                    ; 3
    (print (smaller 2.5 1.5))                                                ; 1.500000
    (print (smaller-by (fn (x y) (> x y)) 3 5))                              ; custom ordering: 5
    (print (smaller-by (fn (x y) (< (Ord.compare x y) 0)) (Some 2) (Some 1))) ; Some(1)
    0))
```

## Notes

- Strings order by bytes (`strcmp`), not locale.
- Mixing Int and Float in one operation is `E_TYPE_MISMATCH` ([fn-int-float-separation](fn-int-float-separation.md)).
- An Int instance of a generic `+ - *` is checked (`E_OVERFLOW`) unless the file declares `(numeric wrapping)` or `(numeric saturating)`, and an Int `/` or `%` needs a nonzero literal divisor or `div!`/`div?` ([fn-numeric-policy](fn-numeric-policy.md), [fn-division-total](fn-division-total.md)).

## See Also

- [fn-string-equality](fn-string-equality.md)
- [trait-static-dispatch](trait-static-dispatch.md)
