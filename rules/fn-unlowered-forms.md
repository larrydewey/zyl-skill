# fn-unlowered-forms

> `make-struct` and `make-variant` do not compile: use `(make-Name ...)` and the constructor. `alias` is a transparent alias and `with-resource` releases its resource (both since 2026-09-28), and `read-line`, `exit` and `close` are lowered.

## Why It Matters

Recognizing a form is not implementing it. A form whose *shape* is wrong is `E_MALFORMED_FORM`, and a form the type checker has no rule for is `E_CANNOT_INFER`, but a well-shaped form that the checker knows and ICNF lowering treats as a plain binding still compiles and then skips its promise: the worst kind of failure, because the program *looks* guarded.

| Form | What it actually does | Use instead |
|---|---|---|
| `(make-struct Name ...)` | `E_CANNOT_INFER` ("no type for form not typed") | `(make-Name ...)` |
| `(make-variant (T) V ...)` | `E_CANNOT_INFER` | `(V ...)` |

These used to be on the list and work now:

| Form | Behavior |
|---|---|
| `(alias A T)` | top level; `A` is `T` wherever a type is written (annotations, fields, variant fields, other aliases); an unknown name in `T` is `E_UNKNOWN_TYPE`, and so is an alias defined in terms of itself |
| `(with-resource (n init) body...)` | binds `n`, runs the body, then `(Drop.drop n)` on the way out, normally or before an error propagates (spec 12.9); an `Int` is a file descriptor (`file-close`); implement `Drop` for your own resource types, else `E_TRAIT_NOT_FOUND` |
| `test-suite`, `setup`, `teardown`, `test-property`, `test-compile`, `assert-fail` | implemented (2026-09-28): [test-suites-properties-compile](test-suites-properties-compile.md) |
| `(read-line)` | one line from stdin without its newline (a trailing `\r` is dropped too); `""` at end of input; flushes stdout first (`zyl_read_line`) |
| `(exit code)` | `code` an `Int`; flushes stdout and stderr and ends the process with that status (`zyl_exit`); its type is fresh, so it fits any branch |
| `(close fd)` | the same as `file-close`: an `Int` descriptor, returns an `Int` |
| `assert`, `unwrap` | lowered since 2026-09-24 ([err-no-assert-unwrap](err-no-assert-unwrap.md)) |

Contracts (`requires`/`ensures`/`invariant`, profiles, `checkpoint` rollback, typed `recover` arms) are implemented ([contract-checks-and-profiles](contract-checks-and-profiles.md)), and `derive` generates Show, Debug, Eq, Ord, Hash and Clone impls ([trait-derive-show](trait-derive-show.md)).

## Bad

```lisp
(defstruct Point (x Int) (y Int))
(defn origin () (make-struct Point 0 0))   ; E_CANNOT_INFER: use (make-Point 0 0)
```

## Good

```lisp
(deftype Log (Log String))
(impl Drop Log (defn drop (self) (match self (Log n (print (str-concat "closed " n))))))

(defn main ()
  (with-resource (l (Log "a"))
    (begin (print "using a") 0)))  ; using a, then closed a
```

## Notes

- Background: ICNF lowering turns any form it has no case for into `(IConst 0)`. That fail-soft default hid real bugs (`for`, `spawn`, `with-resource`, and until 2026-09-25 `read-line`, `exit` and `close`, all once lowered to 0). The type checker now catches forms it has no rule for (`E_CANNOT_INFER`), which is why `make-struct` and `make-variant` fail.
- `exit` flushes stdout and ends the process at once: nothing after it runs, and actors are not joined. Returning a status from `main` is the normal way out.
- `spawn` and the channel forms *are* lowered; see [actor-channels-kahn](actor-channels-kahn.md).

## See Also

- [err-no-assert-unwrap](err-no-assert-unwrap.md)
- [icnf-new-form-needs-case](icnf-new-form-needs-case.md) - why these became 0
