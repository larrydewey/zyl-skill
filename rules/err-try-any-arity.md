# err-try-any-arity

> Catching `error` from a call of any arity works now (the 2/4-argument hang was fixed 2026-09-24); remove old workarounds that kept erroring functions at odd arity.

## Why It Matters

Historical note. Until 2026-09-24, an `error` raised inside some functions of two or four parameters could hang the program under `try` instead of reaching the `catch` (typically when the first argument was a nonzero plain integer). The cause: the `try` path popped the saved frame pointer off the stack before running the body, and the first argument's scratch slot of an even-arity call overwrote it. That is fixed; `error` raised in a function of any arity is caught. Code and notes written before the fix may still carry workarounds.

## Bad

```lisp
; workaround from before the fix: a one-argument helper just to raise the error
(defn fail1 (msg) (error msg))
(defn check (a b) (if (> a b) (fail1 "too big") a))
```

## Good

```lisp
(defn check (a b) (if (> a b) (error "too big") a))
(try (check 5 1) (catch e 0))        ; 0
```

## Notes

- `try` still catches only `error` (and panics such as failed assertions and contracts), not `Err` values ([err-try-catches-error-not-err](err-try-catches-error-not-err.md)).
- Prefer `Result` for failures a caller is expected to handle ([err-result-for-expected-failures](err-result-for-expected-failures.md)).

## See Also

- [err-try-catches-error-not-err](err-try-catches-error-not-err.md)
