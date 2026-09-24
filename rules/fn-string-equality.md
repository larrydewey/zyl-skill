# fn-string-equality

> Compare strings with `str-eq` (or `str-equal`) unless both operands are literals, string built-in results, or `String`-annotated parameters.

## Why It Matters

`=`/`==` compare string **contents** only when the code generator knows both operands are strings. On a parameter, struct field, pattern-bound name or polymorphic result it compares **addresses**. Two equal strings built separately compare unequal. In compiler code this even made output depend on allocation order and broke the self-hosting fixed point.

## Bad

```lisp
(defn same? (a b) (= a b))               ; address comparison
(defn level-is (e lvl) (= (struct-get e "level") lvl))
```

## Good

```lisp
(defn same? (a b) (str-eq a b))          ; 1 or 0, by content
(defn same-str? ((a String) (b String)) (= a b))
(defn level-is (e lvl) (if (> (str-eq (struct-get e "level") lvl) 0) 1 0))
```

## Notes

- `str-eq` returns an Int, `1` or `0`: use it as `(> (str-eq a b) 0)` or `(= (str-eq a b) 1)` in conditions when clarity matters.
- `str-eq` lives in `allocator/allocator`; `str-equal` is an inlined built-in with the same result.
- Literal-pattern `match` compares string alternatives by content.
- Secret data: use `ct-eq-words`, never `=` or an early-exit loop.
- No `+` for strings: use `str-concat`.

## See Also

- [fn-annotate-string-float-params](fn-annotate-string-float-params.md) - why kind matters
- [secret-ct-eq-words-for-bytes](secret-ct-eq-words-for-bytes.md) - constant-time comparison
- [pass-string-eq-in-compiler](pass-string-eq-in-compiler.md) - fixed-point impact
