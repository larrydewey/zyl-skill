# gen-operators-follow-types

> Use `=`, `<` and arithmetic directly in generic code: operators follow the instance's types (Strings compare by content, `<` on Strings is byte order, Floats use SSE); pass comparators only for orderings the operators don't provide.

## Why It Matters

A generic body that applies an operator to a type parameter is instantiated per argument-type tuple ([gen-per-type-instances](gen-per-type-instances.md)), so `(defn smaller (a b) (if (< a b) a b))` compares numbers for Ints and text for Strings. On two records, `==`/`<` compare one level deep ([data-equality-shallow](data-equality-shallow.md)) when the operands are known records; inside an instance they are.

## Good

```lisp
(defn smaller (a b) (if (< a b) a b))
(smaller "banana" "apple")        ; apple
(smaller 3 5)                     ; 3

(defn smaller-by (lt a b) (if (lt a b) a b))
(smaller-by (fn (x y) (> x y)) 3 5)   ; custom ordering: 5
```

## Notes

- Strings order by `strcmp` bytes, not locale.
- Mixing Int and Float in one operation is still unchecked ([fn-int-float-separation](fn-int-float-separation.md)).

## See Also

- [fn-string-equality](fn-string-equality.md)
