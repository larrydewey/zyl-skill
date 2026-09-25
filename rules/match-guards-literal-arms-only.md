# match-guards-literal-arms-only

> Use `(when cond)` guards only after plain literal alternatives; test constructor fields inside the arm body instead.

## Why It Matters

Guards work in exactly one position. Elsewhere they are compile-time errors, with messages that do not mention guards:

| Position | Result |
|---|---|
| After a plain literal: `(0 (when verbose) "...")` | works; the condition must be `Bool` |
| After a `range`: `((range 4 9) (when v) ...)` | `E_ARITY_MISMATCH` (mistaken for the prelude's 2-argument `when` function) |
| On a constructor arm: `(Some x (when (> x 0)) x)` | `E_NESTED_PATTERN` (the guard sits in a binder position) |
| On the trailing `_` arm | `E_MATCH_NONEXHAUSTIVE` in a literal match, `E_NESTED_PATTERN` in a constructor match |

Guards can only refer to names bound outside the `match` (literal arms bind nothing), including the scrutinee's own variable. A guard naming a top-level `def` is mis-handled (`E_UNBOUND_VARIABLE`).

## Bad

```lisp
(match o
  (Some x (when (> x 0)) x)     ; E_NESTED_PATTERN
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
