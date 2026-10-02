# fn-string-equality

> `=`/`==`/`!=` on Strings compare contents and `<`/`>`/`<=`/`>=` order them by bytes, everywhere: every expression has a static type, so there is no address-comparison fallback. Comparing a String with anything else is `E_TYPE_MISMATCH`.

## Why It Matters

Codegen routes String comparisons through `zyl_cstr_eq` / `zyl_cstr_cmp` when the operands' type is String. Since sound type checking became the default (2026-09-25) every value has a type the checker proved ([fn-types-drive-codegen](fn-types-drive-codegen.md)): a value used as both Int and String, or an untyped foreign result, is a compile error, not a word that `=` compares by address. So `(= a b)` on two equal Strings built separately is true in generic helpers, struct fields, pattern binders and container elements alike. The old advice to reach for `str-eq` "when the type is unknown" no longer applies: there is no unknown.

## Bad

Each definition is its own error.

```lisp
(defn cmp () (= 1 "a"))          ; E_TYPE_MISMATCH: the operands of `=` must have one type
(defn join () (+ "a" "b"))       ; E_TYPE_MISMATCH: arithmetic on String; use str-concat
```

## Good

```lisp
(defstruct Ev (level String))
(defn same? (a b) (= a b))                       ; instantiated per type: content compare for Strings
(defn level-is (e lvl) (= (Ev.level e) lvl))

(defn main ()
  (begin
    (print (same? "ab" (str-concat "a" "b")))                 ; 1 (true)
    (print (level-is (make-Ev "warn") (str-concat "wa" "rn"))) ; 1
    (print (< "abc" "abd"))                                   ; 1
    0))
```

`(if (str-eq a b) 1 0)` works too, but `(= a b)` says the same thing.

## Notes

- `str-eq` and `str-equal` take two Strings and return a `Bool`; use them where a String-only signature documents intent.
- Literal-pattern `match` compares string alternatives by content.
- ADTs compare structurally with `=`; ordering an ADT needs `Ord.compare` (`<` on an ADT is `E_TYPE_MISMATCH`, "ordering on T").
- Secret data: use `ct-eq-words`, never `=` or an early-exit loop.
- No `+` for strings: use `str-concat`.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [secret-ct-eq-words-for-bytes](secret-ct-eq-words-for-bytes.md) - constant-time comparison
- [pass-string-eq-in-compiler](pass-string-eq-in-compiler.md) - fixed-point impact
