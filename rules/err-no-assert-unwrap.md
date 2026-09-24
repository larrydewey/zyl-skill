# err-no-assert-unwrap

> `assert` shows a string-literal message (`PANIC: msg`), otherwise a fixed `assert failed`; `unwrap` panics with `unwrap on None` even for an `Err`. Use literal messages and the `-expect` helpers where the message matters.

## Why It Matters

Both were no-ops until 2026-09-24 (`assert` checked nothing, `unwrap` evaluated to 0), and old code or notes may still assume that. Today, verified:

| Form | Success | Failure |
|---|---|---|
| `(assert c "msg")` | 0 | `PANIC: msg`, exit 1 |
| `(assert c)`, or a non-literal message | 0 | `PANIC: assert failed`, exit 1 |
| `(assert-true c "msg")` (outside a test) | | `PANIC: msg`; without a literal message `assert-true failed` |
| `(unwrap (Some v))`, `(unwrap (Ok v))` | `v` | — |
| `(unwrap None)`, `(unwrap (Err e))` | — | `PANIC: unwrap on None`, exit 1, for both |

There is still no `E_ASSERT_FAIL` code. All of these unwind to the nearest `try`, and inside a `test` they fail that test.

## Bad

```lisp
(assert (> n 0) (str-concat "bad n: " name))   ; not a literal: says only "assert failed"
(let v (unwrap (parse s)) (* v 2))             ; an Err reports "unwrap on None"
```

## Good

```lisp
(assert (> n 0) "n must be positive")          ; PANIC: n must be positive
(if (<= n 0) (error (str-concat "bad n: " name)) 0)   ; computed message
(let v (result-expect (parse s) "bad number") (* v 2))
(let v (option-unwrap (lookup k) -1) ...)      ; with a default
```

## Notes

- Inside a `test`, a failure is reported only as `FAIL` ([test-read-summary-line](test-read-summary-line.md)).
- `assert-fail` evaluates its expression and always passes.
- For function pre/postconditions, `requires`/`ensures` give a message naming the function ([contract-not-enforced](contract-not-enforced.md)).

## See Also

- [fn-unlowered-forms](fn-unlowered-forms.md)
- [test-assert-equal-semantics](test-assert-equal-semantics.md)
