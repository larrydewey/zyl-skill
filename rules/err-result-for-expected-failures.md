# err-result-for-expected-failures

> Return `Result` (`Ok`/`Err`) for failures a caller can handle, `Option` (`Some`/`None`) for absence; reserve `error` for unrecoverable conditions.

## Why It Matters

Errors are values in Zyl. There is no null, no exceptions for control flow, and no `?` operator. `error` aborts the program (`PANIC: msg`, exit 1) unless a `try` intercepts it.

## Good

```lisp
(defn divide (a b)
  (if (== b 0) (Err "division by zero") (Ok (/ a b))))

(defn show (r)
  (match r
    (Ok v (print v))
    (Err msg (print msg))))          ; msg has the field's type, String

(deftype ParseErr (Empty) (BadDigit Int) (TooBig))
(defn parse (s) ...)                 ; (Err (BadDigit 3)) -- your own error ADT
```

## Choosing

| Situation | Use |
|---|---|
| Expected failure (parse, lookup) | `Result`; `match` or `result-and-then` |
| Maybe-a-value | `Option`; `match` or `option-map` |
| Default on failure | `result-unwrap r default` / `option-unwrap o default` |
| Turn `Err` into an abort deliberately | `result-expect r "msg"` / `option-expect o "msg"` |
| Unrecoverable condition, internal invariant | `(error "msg")` |
| Several error kinds | `Result` whose `Err` holds your own ADT |

## Notes

- Chain steps with nested `match`, or `(result-and-then r f)` (Result first, function second).
- Conversions: `option-to-result opt err`, `result-to-option res`.

## See Also

- [err-helper-argument-order](err-helper-argument-order.md)
- [err-try-catches-error-not-err](err-try-catches-error-not-err.md)
