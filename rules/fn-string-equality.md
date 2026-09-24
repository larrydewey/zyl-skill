# fn-string-equality

> `=`/`!=` on two Strings compare contents and `<`/`>` order them by bytes wherever their type is known (literals, fields, parameters, container elements, generic instances); use `str-eq` for values whose type may be conflicting or unknown.

## Why It Matters

Codegen routes String comparisons through `zyl_cstr_eq` / `zyl_cstr_cmp` when the operands' kind is String, and inference now supplies that kind almost everywhere ([fn-types-drive-codegen](fn-types-drive-codegen.md)). Where inference gives up (a value used as both Int and String, data mixing types, an untyped FFI result), `=` falls back to comparing **addresses**, and two equal strings built separately compare unequal.

## Good

```lisp
(defn same? (a b) (= a b))                       ; instantiated per type: content compare for Strings
(defn level-is (e lvl) (= (struct-get e "level") lvl))
(same? "ab" (str-concat "a" "b"))                ; 1
(str-eq raw-ffi-result "x")                      ; explicit when the type is unknown
```

## Notes

- `str-eq` returns an Int, `1` or `0`; `str-equal` is an inlined built-in with the same result.
- Literal-pattern `match` compares string alternatives by content.
- Secret data: use `ct-eq-words`, never `=` or an early-exit loop.
- No `+` for strings: use `str-concat`.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [secret-ct-eq-words-for-bytes](secret-ct-eq-words-for-bytes.md) - constant-time comparison
- [pass-string-eq-in-compiler](pass-string-eq-in-compiler.md) - fixed-point impact
