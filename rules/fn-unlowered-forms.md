# fn-unlowered-forms

> Do not use forms that parse but are not lowered: `assert`, `unwrap`, `read-line`, `exit`, `close`, `make-struct`, `make-variant`, `invariant`, and `with-resource` cleanup.

## Why It Matters

Recognizing a form is not implementing it. These compile without complaint and then do nothing or evaluate to 0 — the worst kind of failure, because the program *looks* guarded.

| Form | What it actually does | Use instead |
|---|---|---|
| `(assert c "msg")` | nothing, whatever `c` is | `(if (not c) (error "msg") 0)`; `assert-true` in tests |
| `(unwrap x)` | evaluates to 0 | `result-expect`/`option-expect`, or `-unwrap` with a default |
| `(read-line)` | 0 | not available; read files with `file-read` |
| `(exit code)` | does not end the process | return from `main`; `error` to abort |
| `(close h)` | nothing | `file-close` |
| `(make-struct Name ...)` | 0 | `(make-Name ...)` |
| `(make-variant (T) V ...)` | 0 | `(V ...)` |
| `(invariant c)` | `E_UNBOUND_VARIABLE` (undefined function) | explicit check |
| `(with-resource (n init) body)` | binds `n`; runs **no** release step | release explicitly |
| `test-suite`, `setup`, `teardown`, `test-property`, `test-compile`, `assert-fail` | see [test-unimplemented-features](test-unimplemented-features.md) | flat `test` forms |
| `requires`/`ensures`/`recover`/`checkpoint`/`contracts` | see [contract-not-enforced](contract-not-enforced.md) | explicit checks |

## Bad

```lisp
(defn withdraw (bal amt)
  (begin
    (assert (>= bal amt) "insufficient")   ; checks nothing
    (- bal amt)))
```

## Good

```lisp
(defn withdraw (bal amt)
  (if (< bal amt)
    (error "insufficient funds")
    (- bal amt)))
```

## Notes

- Background: ICNF lowering turns any form it does not recognize into `(IConst 0)`. That fail-soft default has hidden real bugs (`for`, `spawn`, `with-resource` once all lowered to 0).
- `spawn` and `send` *are* lowered, but see [actor-send-is-discarded](actor-send-is-discarded.md).

## See Also

- [err-no-assert-unwrap](err-no-assert-unwrap.md)
- [icnf-new-form-needs-case](icnf-new-form-needs-case.md) - why these become 0
