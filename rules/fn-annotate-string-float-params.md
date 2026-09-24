# fn-annotate-string-float-params

> Annotate parameters that hold a `String` or `Float` — `(s String)`, `(x Float)` — whenever the function prints, compares or does arithmetic on them.

## Why It Matters

Code generation chooses how to print, compare and do arithmetic from a static *kind* (word, String, Float, variant) that it reads off literals and annotations only. An **unannotated parameter, a pattern-bound name, a struct field, a captured variable, and the result of a polymorphic or indirect call are all treated as Int**. Consequences, all silent:

- `print` of a string shows its address (a large integer).
- `print` of a float shows its bit pattern as an integer.
- `(* w h)` on two unannotated Floats multiplies their bit patterns as integers.
- `=` on two unannotated strings compares addresses.

## Bad

```lisp
(defn shout (s) (print (str-concat s "!")))   ; OK: str-concat knows s is a string
(defn show (s) (print s))                      ; prints an address
(defn half (x) (/ x 2.0))                      ; integer-divides the bit pattern
(defn area (s) (match s (Rect w h (* w h))))   ; Float fields multiplied as ints
```

## Good

```lisp
(defn show ((s String)) (print s))
(defn half ((x Float)) (/ x 2.0))
(defn rect-area ((w Float) (h Float)) (* w h))
(defn area (s) (match s (Rect w h (rect-area w h))))

;; or print through the typed core printers:
(print-string (struct-get person "name"))
(print-float (area (Rect 1.5 2.0)))
(print-int (first-of 5 "s"))
```

## Notes

- The core prelude's `print-int`, `print-float`, `print-string`, `print-bool` take annotated parameters and are the standard fix at a call site.
- Passing such values *into* a typed function (`str-concat`, a `defn` with `(s String)`) always works; only printing/comparing/arithmetic where the kind is unknown goes wrong.
- A float literal's kind is known, so `(print 3.14)` prints `3.140000` (six decimals).

## See Also

- [type-polymorphic-results-typed-printers](type-polymorphic-results-typed-printers.md) - same issue through generic calls
- [data-field-kinds](data-field-kinds.md) - struct/ADT fields
- [cg-kind-of](cg-kind-of.md) - the mechanism
