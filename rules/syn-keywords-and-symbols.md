# syn-keywords-and-symbols

> Use `:keyword` tokens only where a form expects them; they are not values.

## Why It Matters

A bare `:foo` in expression position is `E_UNBOUND_VARIABLE` ("`:foo` is not defined"). Keywords are only meaningful inside particular forms. The lexer also skips whitespace after the colon, so `: Ord` and `:Ord` are the same token. The specified `((T : Ord) a b)` and `(a : Int)` spellings are therefore not annotations; since 2026-09-25 both are rejected as `E_MALFORMED_PARAMETER` ("a parameter's type is written (name Type), without a colon") instead of being read as an extra parameter or an ignored annotation.

## Bad

Each definition is its own error.

```lisp
(defn fast () (let mode :fast mode))  ; E_UNBOUND_VARIABLE: `:fast` is not defined
(defn smallest ((T : Ord) a b) a)     ; E_MALFORMED_PARAMETER
(defn f ((a : Int)) a)                ; E_MALFORMED_PARAMETER: write (a Int)
```

## Good

```lisp
(defstruct+ Pt (x) (y) (:derive [Eq Ord]))
(defn f ((a Int)) a)

(defn main ()
  (let b (bytebuf Heap 8)
    (begin
      (store-u8 :le b 0 7)             ; endian selector of byte forms
      (print (load-u8 :le b 0))        ; 7
      (print (== (make-Pt 1 2) (make-Pt 1 2)))  ; 1
      (print (f 3))                    ; 3
      0)))
```

`(use acme/pkg :unsafe { f })` is the import marker (parsed, currently ignored); it needs `acme/pkg` in the package's manifest, so it cannot be shown in a lone file.

## Notes

- Where keywords appear: byte load/store endianness (`:le`/`:be`), `:derive` in `defstruct+`, `:unsafe` in `use`, the options of `with-region` (`:block`, `:align`, `:limit`, `:size`), and `package:module` paths.
- To pass a mode as a value, use an ADT (`(deftype Mode (Fast) (Slow))`) or a Bool; there is no symbol type, and `'fast` is `E_MALFORMED_FORM` ([syn-list-literals-and-quote](syn-list-literals-and-quote.md)).
- `~name` is a SYMBOL token that the reader turns into the plain identifier `name`.
- Identifiers may start with a letter or `_ - ? ! + / = < > * % &` and continue with those, digits and `.`, so `Trait.method`, `set!`, `even?`, `<=`, `&rest` are all single identifiers. `-5` is a number; `-x` is an identifier. `@` is never an identifier character (canonical keys use it).

## See Also

- [gen-no-type-parameter-syntax](gen-no-type-parameter-syntax.md) - the `: Ord` trap
- [fn-no-return-type-slot](fn-no-return-type-slot.md) - the parameter shapes that are accepted
- [syn-naming-conventions](syn-naming-conventions.md) - how names should look
