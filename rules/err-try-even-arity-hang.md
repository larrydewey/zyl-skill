# err-try-even-arity-hang

> Do not rely on catching an `error` raised inside a function called with an even number of arguments; prefer `Result` for failures you expect to handle.

## Why It Matters

Known bug: an `error` raised inside some user functions of **two or four** parameters can **hang** the program instead of reaching the `catch`. It depends on argument values too: with `(defn f (a b) (error "x"))`, `(f 1 2)`, `(f 1 0)` and `(f 1 "a")` hang under `try`, while `(f 0 1)`, `(f "a" 1)` and `(f (Some 1) 2)` are caught — in tests it hangs when the first argument is a nonzero plain integer. Functions of 0, 1 or 3 arguments were unaffected.

## Bad

```lisp
(defn check (a b) (if (> a b) (error "too big") a))
(try (check 5 1) (catch e 0))        ; may hang forever
```

## Good

```lisp
(defn check (a b) (if (> a b) (Err "too big") (Ok a)))
(match (check 5 1) (Ok v v) (Err _ 0))
```

## Notes

- If you must use `try`, keep the erroring function at 1 or 3 parameters, or have the `error` raised from a one-argument helper.
- Uncaught `error` (no `try`) is unaffected: it panics and exits 1.

## See Also

- [err-try-catches-error-not-err](err-try-catches-error-not-err.md)
