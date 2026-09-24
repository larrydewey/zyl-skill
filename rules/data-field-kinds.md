# data-field-kinds

> Treat every struct field and pattern-bound name as an untyped word: route `String` and `Float` values through typed functions before printing, comparing or doing arithmetic.

## Why It Matters

A field is always one 8-byte word — an integer, a float's bit pattern, a bool, or a pointer. Code generation does not know which: pattern-bound names and `struct-get` results are handled as Int. So `(print (struct-get alice "name"))` prints an address, and `(* w h)` on two pattern-bound Floats multiplies bit patterns.

## Bad

```lisp
(deftype Shape (Circle Float) (Rect Float Float))
(defn area (s)
  (match s
    (Circle r (* 3.14 r r))      ; r treated as Int: wrong
    (Rect w h (* w h))))         ; bit patterns multiplied
(print (area (Rect 1.5 2.0)))    ; garbage integer
```

## Good

```lisp
(defn circle-area ((r Float)) (* 3.14 r r))
(defn rect-area ((w Float) (h Float)) (* w h))
(defn area (s)
  (match s
    (Circle r (circle-area r))
    (Rect w h (rect-area w h))))
(print-float (area (Rect 1.5 2.0)))   ; 3.000000
```

## Notes

- For strings: `print-string`, `str-eq`, or a `(s String)` parameter.
- Prefer Int fields (fixed-point) where you can; the book's shape examples use Int for exactly this reason.

## See Also

- [fn-annotate-string-float-params](fn-annotate-string-float-params.md)
- [cg-kind-of](cg-kind-of.md)
