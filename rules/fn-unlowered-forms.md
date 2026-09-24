# fn-unlowered-forms

> Do not use forms that parse but are not lowered: `read-line`, `exit`, `close`, `make-struct`, `make-variant`, `invariant`, and `with-resource` cleanup.

## Why It Matters

Recognizing a form is not implementing it. These compile without complaint and then do nothing or evaluate to 0 — the worst kind of failure, because the program *looks* guarded.

| Form | What it actually does | Use instead |
|---|---|---|
| `(read-line)` | 0 | not available; read files with `file-read` |
| `(exit code)` | does not end the process | return from `main`; `error` to abort |
| `(close h)` | nothing | `file-close` |
| `(make-struct Name ...)` | 0 | `(make-Name ...)` |
| `(make-variant (T) V ...)` | 0 (matching it segfaults) | `(V ...)` |
| `(invariant c)` | `E_UNBOUND_VARIABLE` (undefined function) | explicit check |
| `(with-resource (n init) body)` | binds `n`; runs **no** release step | release explicitly |
| `test-suite`, `setup`, `teardown`, `test-property`, `test-compile`, `assert-fail` | see [test-unimplemented-features](test-unimplemented-features.md) | flat `test` forms |
| `requires`/`ensures`/`recover`/`checkpoint`/`contracts` | see [contract-not-enforced](contract-not-enforced.md) | explicit checks |

`assert` and `unwrap` were on this list until 2026-09-24; they are lowered now but lose their messages ([err-no-assert-unwrap](err-no-assert-unwrap.md)).

## Bad

```lisp
(defn withdraw (bal amt)
  (if (< bal amt)
    (exit 1)                               ; does not end the process; evaluates to 0
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
