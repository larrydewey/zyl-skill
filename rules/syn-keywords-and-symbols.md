# syn-keywords-and-symbols

> Use `:keyword` tokens only where a form expects them; they are not values.

## Why It Matters

A bare `:foo` in expression position is `E_UNBOUND_VARIABLE`. Keywords are only meaningful inside particular forms. The lexer also skips whitespace after the colon, so `: Ord` and `:Ord` are the same token, which is how the specified `((T : Ord) a b)` syntax turns into an extra value parameter.

## Bad

```lisp
(let mode :fast (print mode))       ; E_UNBOUND_VARIABLE
(defn smallest ((T : Ord) a b) ...) ; (T :Ord) is a VALUE parameter named T
```

## Good

```lisp
(load-u8 :le buf 0)                  ; endian selector of byte forms
(defstruct+ Pt (x) (y) (:derive [Eq Ord]))
(use acme/pkg :unsafe { f })         ; import marker (parsed, currently ignored)
```

## Notes

- Where keywords appear: byte load/store endianness (`:le`/`:be`), `:derive` in `defstruct+`, `:unsafe` in `use`, and `package:module` paths.
- `~name` is a SYMBOL token that the reader turns into the plain identifier `name`.
- Identifiers may contain `- ? ! + / = < > * %` and `.` after the first character, so `Trait.method`, `set!`, `even?`, `<=` are all single identifiers. `-5` is a number; `-x` is an identifier. `@` is never an identifier character (canonical keys use it).

## See Also

- [gen-no-type-parameter-syntax](gen-no-type-parameter-syntax.md) - the `: Ord` trap
- [syn-naming-conventions](syn-naming-conventions.md) - how names should look
