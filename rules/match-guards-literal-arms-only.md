# match-guards-literal-arms-only

> Use `(when cond)` guards only after plain literal alternatives; test constructor fields inside the arm body instead.

## Why It Matters

Guards work in exactly one position. Elsewhere:

| Position | Result |
|---|---|
| After a plain literal: `(0 (when verbose) "...")` | works |
| After a `range`: `((range 4 9) (when v) ...)` | `E_ARITY_MISMATCH` (mistaken for the prelude's 2-arg `when` function) |
| On a constructor arm: `(Some x (when (> x 0)) x)` | read as a field binder: **compiles, then crashes at run time** |
| On the trailing `_` arm | silently ignored |

Guards can only refer to names bound outside the `match` (literal arms bind nothing).

## Bad

```lisp
(match o
  (Some x (when (> x 0)) x)     ; compiles; crashes
  (_ 0))
```

## Good

```lisp
(defn positive-or-zero (o)
  (match o
    (Some x (if (> x 0) x 0))
    (None 0)))

(defn zero-word (n verbose)
  (match n
    (0 (when verbose) "zero, exactly")
    (0 "zero")
    (_ "nonzero")))
```

## See Also

- [match-literal-requires-underscore](match-literal-requires-underscore.md)
- [match-no-nested-patterns](match-no-nested-patterns.md)
