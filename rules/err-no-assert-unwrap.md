# err-no-assert-unwrap

> `assert` and `unwrap` work but lose information: `assert` panics with a fixed `assert failed`, and `unwrap` panics with `unwrap on None` even for an `Err`. Prefer `error` and the `-expect` helpers where the message matters.

## Why It Matters

Both were no-ops until 2026-09-24 (`assert` checked nothing, `unwrap` evaluated to 0), and old code or notes may still assume that. Today, verified:

| Form | Success | Failure |
|---|---|---|
| `(assert c "msg")` | 0 | `PANIC: assert failed`, exit 1; your `"msg"` is dropped; no `E_ASSERT_FAIL` code |
| `(unwrap (Some v))`, `(unwrap (Ok v))` | `v` | — |
| `(unwrap None)`, `(unwrap (Err e))` | — | `PANIC: unwrap on None`, exit 1, for both |

Both unwind to the nearest `try`, and inside a `test` they fail that test.

## Bad

```lisp
(assert (> n 0) "n must be positive")          ; failure says only "assert failed"
(let v (unwrap (parse s)) (* v 2))             ; an Err reports "unwrap on None"
```

## Good

```lisp
(if (<= n 0) (error "n must be positive") 0)
(let v (result-expect (parse s) "bad number") (* v 2))
(let v (option-unwrap (lookup k) -1) ...)      ; with a default
```

## Notes

- In tests, `assert-true`, `assert-false` and `assert-equal` report better (outside a test they panic and exit 1 on failure).
- `assert-fail` evaluates its expression and always passes.

## See Also

- [fn-unlowered-forms](fn-unlowered-forms.md)
- [test-assert-equal-semantics](test-assert-equal-semantics.md)
