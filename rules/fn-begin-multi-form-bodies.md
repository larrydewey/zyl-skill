# fn-begin-multi-form-bodies

> Wrap every multi-step `if` branch and `match` arm body in `begin`; other bodies (`defn`, `let`, `fn`, `while`, `for`, `cond` clauses, `catch` handlers) already sequence their forms, while `test` and `defmacro` take exactly one.

## Why It Matters

Since 2026-09-25 bodies that take several forms are real implicit `begin`s, scoped as the language defines: a `let` that is one form of a body no longer leaks into the forms after it (it used to replace the outer name for the rest of the body), and `fn`, `catch` handlers and `cond` clauses no longer keep only their last form.

The places that take exactly one expression are where multi-step code still goes wrong:

- `if` takes a condition and at most two branches. A form after the else branch is `E_MALFORMED_FORM` (since 2026-09-25; it used to be dropped silently).
- A `match` arm is a pattern followed by one body. Extra forms are read as part of the pattern, so `(Some n (print n) n)` is `E_NESTED_PATTERN` (the message talks about a constructor field, not about the missing `begin`).
- `test` and `defmacro` take exactly one body form; more is `E_MALFORMED_FORM` ("malformed `test` form").

## Bad

```lisp
(if (> x 0)
  (print "positive")
  (print "not positive")
  (print "done"))                 ; E_MALFORMED_FORM: a form after the else branch

(match opt
  (Some n (print n) n)            ; E_NESTED_PATTERN
  (None 0))

(defmacro two (a b) (print a) (print b))   ; E_MALFORMED_FORM
```

## Good

```lisp
(begin
  (if (> x 0)
    (print "positive")
    (print "not positive"))
  (print "done"))

(match opt
  (Some n (begin (print n) n))
  (None 0))

(defmacro two (a b) (begin (print a) (print b)))

(defn main ()
  (let x 10
    (let x 20 (print x))          ; 20 (W_SHADOWED_BINDING)
    (print x)                     ; 10: the inner let does not leak
    0))
```

## Notes

- `(begin e1 ... en)` evaluates left to right; its value is `en`. An empty `(begin)` is the Unit value, like `unit`.
- Every form but the last in a body is a statement; its value is discarded, so a non-Unit value there is fine.
- A macro with several steps puts them in one `begin`; with a `&rest` body, splice it: `(begin ,@body)` ([macro-quasiquote-and-rest](macro-quasiquote-and-rest.md)).

## See Also

- [fn-let-single-binding](fn-let-single-binding.md) - the other `let` trap
- [fn-conditionals](fn-conditionals.md) - `if` shapes and Bool conditions
- [match-arm-shape](match-arm-shape.md) - one pattern, one body
