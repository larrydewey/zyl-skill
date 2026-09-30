# err-result-for-expected-failures

> Return `Result` (`Ok`/`Err`) for failures a caller can handle, `Option` (`Some`/`None`) for absence; reserve `panic` for unrecoverable conditions.

## Why It Matters

Errors are values in Zyl. There is no null, no exceptions for control flow, and no `?` operator. The two halves are separate functions, and the names are easy to swap:

| Form | Does | For |
|---|---|---|
| `(error "msg")` | returns `(Err "msg")`, raises nothing | a failure the caller should handle — match on it |
| `(panic "msg")` | unwinds to the nearest `try`, else `PANIC: msg` and exit 1 | an unrecoverable condition, an internal invariant |

`error` lives in `core/result` beside the `Err` it builds; `panic` in `allocator`. Swapping the names is caught by the type checker wherever a value of the other shape is expected — `(if (> b 0) b (error "one"))` is `E_TYPE_MISMATCH` — but not in statement position, where the value is discarded: `(begin (error "x") (print "after"))` compiles, prints, and exits 0. When a program that should abort carries on, look for `error` written where `panic` was meant.

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
| Turn `Err` into a panic deliberately | `result-expect r "msg"` / `option-expect o "msg"` |
| Unrecoverable condition, internal invariant | `(panic "msg")` |
| Several error kinds | `Result` whose `Err` holds your own ADT |

## Notes

- `Result` is `(Result T E)` and checked: every `Ok` a function returns has one type, and so does every `Err` (`(if c (Err "x") (Err 3))` is `E_TYPE_MISMATCH`). For several error kinds, make `E` your own ADT. An error ADT without a `Show` impl prints as an address inside `Err(...)`: derive `Show`.
- Chain steps with nested `match`, or `(result-and-then r f)` (Result first, function second).
- Conversions: `option-to-result opt err`, `result-to-option res`.
- `try` cannot stand in for handling an `Err`: the body and handler must share one type, so `(try (error "x") (catch e 7))` is `E_TYPE_MISMATCH`. See [err-try-catches-panic-not-err](err-try-catches-panic-not-err.md).

## See Also

- [err-helper-argument-order](err-helper-argument-order.md)
- [err-try-catches-panic-not-err](err-try-catches-panic-not-err.md)
