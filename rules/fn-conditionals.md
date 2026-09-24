# fn-conditionals

> Always give `if` an else branch and `cond` an `else` clause in value position; conditions must be real Bools.

## Why It Matters

A one-armed `if` whose condition is false evaluates to 0, and a `cond` with no true test evaluates to 0. That is fine for effects and a silent wrong value anywhere else. There is no truthiness beyond Bool: write the comparison.

## Bad

```lisp
(defn sign (n) (if (> n 0) 1))             ; 0 for n <= 0 -- intended?
(defn classify (n) (cond ((< n 0) "neg"))) ; 0 (not a string!) otherwise
```

## Good

```lisp
(defn sign (n)
  (if (> n 0) 1
    (if (< n 0) -1 0)))

(defn classify (n)
  (cond
    ((< n 0) "negative")
    ((== n 0) "zero")
    (else "positive")))

(defn log-if-positive (n)
  (if (> n 0) (print n)))                  ; effect-only: no else is fine
```

## Notes

- `and`/`or` are desugared to `if` and short-circuit. `(or 5 6)` yields `true` (1), not 5: a non-final operand that succeeds yields `true`; the last operand is returned as is (`(or false 7)` is 7).
- `when`/`unless` in the prelude are **functions**: both arguments are evaluated. `(when false (print "x"))` prints. Use `if` for effects.
- Comparisons: `=`/`==` (same op), `!=`, `<`, `>`, `<=`, `>=`.

## See Also

- [macro-prefer-functions](macro-prefer-functions.md) - lazy `when` as a macro
- [match-literal-requires-underscore](match-literal-requires-underscore.md) - multi-way on values
