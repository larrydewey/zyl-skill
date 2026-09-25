# err-no-assert-unwrap

> `assert` takes a `Bool` and shows a string-literal message (`PANIC: msg`), otherwise a fixed `assert failed`; `unwrap` takes only an `Option` and panics with `unwrap on None`. Use literal messages and the `-expect` helpers where the message matters.

## Why It Matters

Both were no-ops until 2026-09-24 (`assert` checked nothing, `unwrap` evaluated to 0), and old code or notes may still assume that. Today, verified:

| Form | Success | Failure |
|---|---|---|
| `(assert c "msg")` | `unit` | `PANIC: msg`, exit 1 |
| `(assert c)`, or a non-literal message | `unit` | `PANIC: assert failed`, exit 1 |
| `(assert-true c "msg")` (outside a test) | `unit` | `PANIC: msg`; without a literal message `assert-true failed` |
| `(unwrap (Some v))` | `v` | — |
| `(unwrap None)` | — | `PANIC: unwrap on None`, exit 1 |
| `(unwrap (Ok v))`, `(unwrap (Err e))` | `E_TYPE_MISMATCH` at compile time: `unwrap` is for `Option` | |
| `(assert 1 "x")` | `E_TYPE_MISMATCH` at compile time: the condition is `Bool` | |

There is still no `E_ASSERT_FAIL` code. All of these unwind to the nearest `try`, and inside a `test` they fail that test.

## Bad

```lisp
(assert (> n 0) (str-concat "bad n: " name))   ; not a literal: says only "assert failed"
(let v (unwrap (parse s)) (* v 2))             ; parse returns a Result: E_TYPE_MISMATCH
```

## Good

```lisp
(assert (> n 0) "n must be positive")                   ; PANIC: n must be positive
(if (<= n 0) (error (str-concat "bad n: " name)) unit)  ; computed message
(let v (result-expect (parse s) "bad number") (* v 2))
(let v (option-unwrap (lookup k) -1) ...)               ; with a default
```

## Notes

- Inside a `test`, a failure is reported only as `FAIL` ([test-read-summary-line](test-read-summary-line.md)).
- `assert-fail` evaluates its expression and always passes.
- For function pre/postconditions, `requires`/`ensures` give a message naming the function ([contract-checks-and-profiles](contract-checks-and-profiles.md)).

## See Also

- [fn-unlowered-forms](fn-unlowered-forms.md)
- [test-assert-equal-semantics](test-assert-equal-semantics.md)
