# fn-unlowered-forms

> Do not use forms that type-check but are not lowered: `read-line`, `exit`, `close`, `with-resource` cleanup, and `alias`; `make-struct` and `make-variant` no longer compile at all.

## Why It Matters

Recognizing a form is not implementing it. Since 2026-09-25 a form whose *shape* is wrong is `E_MALFORMED_FORM` instead of the constant 0, but a well-shaped form that the type checker knows and ICNF lowering does not still compiles and then does nothing: the worst kind of failure, because the program *looks* guarded.

| Form | What it actually does | Use instead |
|---|---|---|
| `(read-line)` | typed `String`, returns a null string (prints `(null)`, length 0) | not available; read files with `file-read` |
| `(exit code)` | type-checks as any type, does not end the process, evaluates to 0 | return from `main`; `error` to abort |
| `(close h)` | nothing | `file-close` |
| `(with-resource (n init) body...)` | binds `n` and runs the body; runs **no** release step | release explicitly |
| `(alias A T)` | nothing; `A` in an annotation is then a fresh type variable, so `(x A)` accepts any type | the original type name |
| `(make-struct Name ...)` | `E_CANNOT_INFER` ("no type for form not typed") | `(make-Name ...)` |
| `(make-variant (T) V ...)` | `E_CANNOT_INFER` | `(V ...)` |
| `test-suite`, `setup`, `teardown`, `test-property`, `test-compile`, `assert-fail` | see [test-unimplemented-features](test-unimplemented-features.md) | flat `test` forms |

`assert` and `unwrap` were on this list until 2026-09-24; they are lowered now; `assert` shows a string-literal message and `unwrap` panics with `unwrap on None` ([err-no-assert-unwrap](err-no-assert-unwrap.md)). Contracts (`requires`/`ensures`/`invariant`, profiles, `checkpoint` rollback, typed `recover` arms) are implemented since the same date ([contract-checks-and-profiles](contract-checks-and-profiles.md)), and `derive` generates Show, Debug, Eq, Ord, Hash and Clone impls ([trait-derive-show](trait-derive-show.md)).

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

- Background: ICNF lowering turns any form it has no case for into `(IConst 0)`. That fail-soft default has hidden real bugs (`for`, `spawn`, `with-resource` once all lowered to 0). The type checker now catches the forms it has no rule for (`E_CANNOT_INFER`), which is why `make-struct` and `make-variant` fail; `read-line`, `exit` and `close` have typing rules but no lowering.
- `(exit 1)` has a fresh type, so it fits any branch: `withdraw` above type-checks and returns 0 for an overdraft.
- `spawn`, `send`, `receive` and `actor-self` *are* lowered; see [actor-send-is-discarded](actor-send-is-discarded.md).

## See Also

- [err-no-assert-unwrap](err-no-assert-unwrap.md)
- [icnf-new-form-needs-case](icnf-new-form-needs-case.md) - why these become 0
