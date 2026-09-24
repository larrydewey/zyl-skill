# fn-no-toplevel-def

> Use a zero-argument `defn` for constants; a top-level `def` is not readable in compiled code.

## Why It Matters

`(def name value)` at top level is accepted, but any reference to `name` from a function is `E_UNBOUND_VARIABLE`. The Global region is not implemented. At the REPL `def` does bind a value, which makes the difference easy to miss when prototyping there.

## Bad

```lisp
(def max-size 1000)
(defn main () (print max-size))   ; E_UNBOUND_VARIABLE
```

## Good

```lisp
(defn max-size () 1000)
(defn app-name () "MyApp")
(defn main ()
  (begin
    (print (max-size))
    (print (app-name))
    0))
```

## Notes

- Costs one call per use; reads almost the same.
- There is no global mutable state at all: every binding is local.

## See Also

- [own-regions-status](own-regions-status.md) - Global region not implemented
- [tool-repl](tool-repl.md) - REPL `def` semantics differ
