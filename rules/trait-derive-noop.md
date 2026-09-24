# trait-derive-noop

> Don't rely on `derive` or `defstruct+ (:derive ...)`: both are accepted and generate nothing.

## Why It Matters

`insert-derive` returns its input unchanged. No impl is generated, no field requirement is checked, unknown trait names are accepted, and `E_TRAIT_NOT_DERIVABLE` is never raised. You get the built-in behavior with or without it:

- `==`, `!=`, `assert-equal`: structural, one level deep ([data-equality-shallow](data-equality-shallow.md)).
- `<`, `>`, `<=`, `>=`: lexicographic over fields, shallow.
- `print`: address (no `Show`/`Debug` text).
- `Clone`, `Hash`: nothing to call.

## Good

```lisp
(defstruct Pt (x) (y))
;; (derive Pt Eq Ord)  -- harmless, changes nothing
(defn pt-show (p)
  (begin (print (struct-get p "x")) (print (struct-get p "y"))))
```

## See Also

- [data-equality-shallow](data-equality-shallow.md)
