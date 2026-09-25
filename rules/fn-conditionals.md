# fn-conditionals

> Conditions must be `Bool`; an `if` without else and a `cond` without `else` are `Unit`, so give both a final branch whenever their value is used.

## Why It Matters

Since 2026-09-25 the type checker enforces what used to be a silent wrong value. A one-armed `if` and a `cond` with no `else` clause have type `Unit`, so using one where an Int or String is expected is `E_TYPE_MISMATCH` ("cannot unify Int with Unit"). Every condition (`if`, `cond`, `while`, `when`, guards, assertions, contract clauses) must be a `Bool`: there is no truthiness, so `(if 1 ...)` or `(if count ...)` is `E_TYPE_MISMATCH` ("cannot unify Int with Bool"). `and`, `or` and `not` take and return Bools.

## Bad

```lisp
(defn sign (n) (if (> n 0) 1))             ; E_TYPE_MISMATCH: Int with Unit
(defn classify (n) (cond ((< n 0) "neg"))) ; E_TYPE_MISMATCH: String with Unit
(if 1 (print "y") (print "n"))             ; E_TYPE_MISMATCH: Int with Bool
(or 5 6)                                   ; E_TYPE_MISMATCH: operands must be Bool
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
  (if (> n 0) (print n)))                  ; Unit on both paths: no else needed
```

## Notes

- A branch that is only an effect (`print`, `set!`, a loop) is `Unit`, so both branches of an `if` in statement position must be Unit or both something else; `(if c (print "x") 0)` is `E_TYPE_MISMATCH`. Use `unit` for an explicit empty branch.
- `if` takes at most two branches: forms after the else branch are silently dropped ([fn-begin-multi-form-bodies](fn-begin-multi-form-bodies.md)).
- `and`/`or` are desugared to `if` and short-circuit.
- `when`/`unless` in the prelude are **functions**: both arguments are evaluated, so `(when false (print "x"))` prints. Their body must be `Unit`. Use `if` for effects, or write a macro ([macro-prefer-functions](macro-prefer-functions.md)).
- A `cond` clause whose test is the literal `true` or `else` ends the `cond`; clauses after it are unreachable. A clause body may hold several forms.
- Comparisons: `=`/`==` (same op), `!=`, `<`, `>`, `<=`, `>=`. Ordering works on Int, Float and String; order an ADT with `Ord.compare`.
- A Bool prints as `1`/`0` with `print`, but as `true`/`false` inside a printed container.

## See Also

- [macro-prefer-functions](macro-prefer-functions.md) - lazy `when` as a macro
- [match-literal-requires-underscore](match-literal-requires-underscore.md) - multi-way on values
