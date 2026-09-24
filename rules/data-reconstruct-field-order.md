# data-reconstruct-field-order

> When rebuilding a struct or variant, pass every field in declaration order — double-check against the `defstruct`/`deftype`.

## Why It Matters

Constructors are positional, and field types catch a swap only when the two types definitely differ (`E_TYPE_MISMATCH`). Rebuilding a record with two same-typed or untyped fields swapped compiles and silently mislabels every downstream use. In the self-hosted compiler, where state records (`CGS`, `ST`, `Stats`...) are rebuilt constantly, a swapped field shows up as "garbage where a variable should be" or a wrong count far from the bug.

## Bad

```lisp
(defstruct Stats (total) (errors) (warnings) (infos) (skipped))
(defn count-skipped (st)
  (make-Stats (struct-get st "total")
              (struct-get st "warnings")   ; swapped with errors
              (struct-get st "errors")
              (struct-get st "infos")
              (+ (struct-get st "skipped") 1)))
```

## Good

```lisp
(defn count-skipped (st)
  (make-Stats (struct-get st "total")
              (struct-get st "errors")
              (struct-get st "warnings")
              (struct-get st "infos")
              (+ (struct-get st "skipped") 1)))

;; ADT state threading: destructure and rebuild in the same order
(deftype ST (ST Int Int))
(defn step (st x)
  (match st
    (ST a b (ST (+ a 1) (+ b x)))))
```

## Notes

- Keep records small, and write one "update one field" helper per field that everything else uses.
- A test that checks each field after an update catches this class of bug cheaply.

## See Also

- [data-struct-immutable-rebind](data-struct-immutable-rebind.md)
- [pass-state-threading](pass-state-threading.md)
