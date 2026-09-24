# type-polymorphic-results-typed-printers

> Print or compare the result of a polymorphic (unannotated) function through a typed function: `print-int`, `print-string`, `print-float`, or a `(x Type)` parameter.

## Why It Matters

Types do not flow back out of a polymorphic call into `print`. When a function with unannotated parameters returns a `String` or `Float`, `print` formats it as an Int (an address or bit pattern). The same applies to results of calls through function values.

## Bad

```lisp
(defn ident (x) x)
(print (ident "s"))          ; prints an address
(print (first-of 2.5 0))     ; prints float bits as an integer
```

## Good

```lisp
(defn ident (x) x)
(print-string (ident "s"))   ; s
(print-int (ident 5))        ; 5
(print-float (ident 2.5))    ; 2.500000
```

## Notes

- `print` on a direct literal or annotated parameter is fine.
- Code generation's `kind-of` has no return-type inference: a function's result kind is read off its body only when the body has a fixed shape; otherwise it is "word".

## See Also

- [fn-annotate-string-float-params](fn-annotate-string-float-params.md)
- [cg-kind-of](cg-kind-of.md)
