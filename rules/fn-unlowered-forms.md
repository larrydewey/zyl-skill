# fn-unlowered-forms

> Do not rely on `with-resource` cleanup or `alias`: they type-check but do nothing; `make-struct` and `make-variant` do not compile at all. `read-line`, `exit` and `close` are lowered since 2026-09-25.

## Why It Matters

Recognizing a form is not implementing it. A form whose *shape* is wrong is `E_MALFORMED_FORM`, and a form the type checker has no rule for is `E_CANNOT_INFER`, but a well-shaped form that the checker knows and ICNF lowering treats as a plain binding still compiles and then skips its promise: the worst kind of failure, because the program *looks* guarded.

| Form | What it actually does | Use instead |
|---|---|---|
| `(with-resource (n init) body...)` | binds `n` and runs the body; runs **no** release step | release explicitly |
| `(alias A T)` | nothing; `A` in an annotation is then a fresh type variable, so `(x A)` accepts any type | the original type name |
| `(make-struct Name ...)` | `E_CANNOT_INFER` ("no type for form not typed") | `(make-Name ...)` |
| `(make-variant (T) V ...)` | `E_CANNOT_INFER` | `(V ...)` |
| `test-suite`, `setup`, `teardown`, `test-property`, `test-compile`, `assert-fail` | see [test-unimplemented-features](test-unimplemented-features.md) | flat `test` forms |

These used to be on the list and work now:

| Form | Behavior |
|---|---|
| `(read-line)` | one line from stdin without its newline (a trailing `\r` is dropped too); `""` at end of input; flushes stdout first (`zyl_read_line`) |
| `(exit code)` | `code` an `Int`; flushes stdout and stderr and ends the process with that status (`zyl_exit`); its type is fresh, so it fits any branch |
| `(close fd)` | the same as `file-close`: an `Int` descriptor, returns an `Int` |
| `assert`, `unwrap` | lowered since 2026-09-24 ([err-no-assert-unwrap](err-no-assert-unwrap.md)) |

Contracts (`requires`/`ensures`/`invariant`, profiles, `checkpoint` rollback, typed `recover` arms) are implemented ([contract-checks-and-profiles](contract-checks-and-profiles.md)), and `derive` generates Show, Debug, Eq, Ord, Hash and Clone impls ([trait-derive-show](trait-derive-show.md)).

## Bad

```lisp
(defn main ()
  (with-resource (fd (file-open "out.txt" "w"))
    (begin
      (file-write fd "data")
      0)))                                 ; fd is never closed
```

## Good

```lisp
(defn main ()
  (let fd (file-open "out.txt" "w")
    (let _ (file-write fd "data")
      (let _ (close fd)
        0))))
```

## Notes

- Background: ICNF lowering turns any form it has no case for into `(IConst 0)`. That fail-soft default hid real bugs (`for`, `spawn`, `with-resource`, and until 2026-09-25 `read-line`, `exit` and `close`, all once lowered to 0). The type checker now catches forms it has no rule for (`E_CANNOT_INFER`), which is why `make-struct` and `make-variant` fail.
- `exit` ends the process at once: nothing after it runs, and actors are not drained. Returning a status from `main` is the normal way out.
- `spawn`, `send`, `receive` and `actor-self` *are* lowered; see [actor-send-is-discarded](actor-send-is-discarded.md).

## See Also

- [err-no-assert-unwrap](err-no-assert-unwrap.md)
- [icnf-new-form-needs-case](icnf-new-form-needs-case.md) - why these became 0
