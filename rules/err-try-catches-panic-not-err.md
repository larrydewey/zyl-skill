# err-try-catches-panic-not-err

> `try`/`catch` intercepts panics only; an `(Err ...)` value passes straight through it, and the type checker rejects trying to catch one.

## Why It Matters

`try` is `catch_unwind`, not `?`. An `Err` is an ordinary value. Code that wraps a `Result`-returning call in `try` expecting to handle the `Err` silently handles nothing.

Since `error` returns an `Err` rather than raising, this is now the difference between two similarly named functions: `(error "x")` gives you a value to match, `(panic "x")` gives `try` something to catch.

## Bad

```lisp
(try (parse-config path)             ; returns (Err "bad")
  (catch e (default-config)))        ; handler never runs; you get (Err "bad")
```

## Good

```lisp
(match (parse-config path)
  (Ok c c)
  (Err _ (default-config)))

(defn percent-of (n)
  (if (== n 0) (panic "division by zero") (/ 100 n)))
(defn report ((msg String)) (begin (print-string (str-concat "caught: " msg)) -1))
(print (try (percent-of 0) (catch e (report e))))   ; caught: division by zero, -1
```

## Notes

- Syntax: `(try expr (catch name handler))`. One handler expression; use `begin` or a function for more.
- `name` is bound to the message **String**, and prints as one.
- The handler has the body's type: `(try (check 5 1) (catch e "s"))` with an Int body is `E_TYPE_MISMATCH`.
- `try` also catches failed assertions (`assert-equal` etc.), `result-expect`/`option-expect` failures, contract violations, `E_INDEX_OUT_OF_BOUNDS` from `vec-get`/views/slices, and catchable runtime errors such as `E_FFI_TIMEOUT` and `E_REGION_EXHAUSTED`.
- Without an active `try`, `panic` prints `PANIC: msg` to stderr and exits 1 (the PANIC line may appear before earlier buffered stdout).
- The handler has the body's type, so `(try (error "x") (catch e 7))` does not even compile: a `Result` body cannot be paired with an `Int` handler. That is deliberate — it is the type checker refusing to let `try` pose as `?`. `match` is the only way to handle an `Err`.

## See Also

- [err-try-any-arity](err-try-any-arity.md)
- [err-result-for-expected-failures](err-result-for-expected-failures.md)
