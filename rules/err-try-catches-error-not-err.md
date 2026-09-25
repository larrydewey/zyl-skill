# err-try-catches-error-not-err

> `try`/`catch` intercepts `error` panics only; an `(Err ...)` value passes straight through it.

## Why It Matters

`try` is `catch_unwind`, not `?`. An `Err` is an ordinary value. Code that wraps a `Result`-returning call in `try` expecting to handle the `Err` silently handles nothing.

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
  (if (== n 0) (error "division by zero") (/ 100 n)))
(defn report ((msg String)) (begin (print-string (str-concat "caught: " msg)) -1))
(print (try (percent-of 0) (catch e (report e))))   ; caught: division by zero, -1
```

## Notes

- Syntax: `(try expr (catch name handler))`. One handler expression; use `begin` or a function for more.
- `name` is bound to the message **String**, and prints as one.
- The handler has the body's type: `(try (check 5 1) (catch e "s"))` with an Int body is `E_TYPE_MISMATCH`.
- `try` also catches failed assertions (`assert-equal` etc.), `result-expect`/`option-expect` failures, contract violations, `E_INDEX_OUT_OF_BOUNDS` from `vec-get`/views/slices, and catchable runtime errors such as `E_FFI_TIMEOUT` and `E_REGION_EXHAUSTED`.
- Without an active `try`, `error` prints `PANIC: msg` to stderr and exits 1 (the PANIC line may appear before earlier buffered stdout).
- The spec describes `error` as returning `(Err msg)`; the implementation panics.

## See Also

- [err-try-any-arity](err-try-any-arity.md)
- [err-result-for-expected-failures](err-result-for-expected-failures.md)
