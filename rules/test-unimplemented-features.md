# test-unimplemented-features

> Don't use `test-suite`, `setup`/`teardown`, `test-property`, `test-compile`, `assert-fail`, keyword options or the `run-tests-*` helpers: they compile and do nothing, silently drop tests, or are rejected.

## Why It Matters

The worst one is `test-suite`: the tests nested inside it are **silently dropped** — neither registered nor run — so a suite looks green while testing nothing. The type checker does not help: these forms are parsed as placeholders that carry no code, so their bodies are never type-checked either.

| Feature | Behavior |
|---|---|
| `(test-suite "name" (test ...) ...)` | nested tests dropped |
| `setup` / `teardown` | accepted at top level, never run |
| `(test-property "n" gen prop)` | compiled to nothing, never run (its arguments are not even checked) |
| `test-compile` | no effect |
| `assert-fail` | always passes |
| `assert` | works: a failing `assert` fails the test (it panics; the message is not shown) |
| a keyword option on `test`: `(test "n" :parallel body)` | `E_MALFORMED_FORM` (a test is a name and one body form) |
| a keyword on `run-tests`: `(run-tests :verbose)` | ignored |
| `run-tests-filtered`, `run-tests-parallel`, `run-tests-with-timeout`, `test-count` (`testing/testing`) | placeholders that call `error` |
| `property-int` & friends (`testing/testing`) | wrap `test-property`, so they do nothing |

## Bad

```lisp
(test-suite "math"
  (test "add" (assert-equal (+ 1 2) 3)))   ; never runs
(run-tests)                                ; test result: 0 passed, 0 failed, 0 total
```

## Good

```lisp
(test "math-add" (assert-equal (+ 1 2) 3))
(run-tests)
```

## Notes

- No import is needed for `test`, the assertions or `run-tests`. `(use testing/testing)` adds only the placeholders above, and its unused parameters print a screenful of `W_UNUSED_PARAMETER` warnings on every compile.

## See Also

- [test-toplevel-forms-and-run-tests](test-toplevel-forms-and-run-tests.md)
- [fn-unlowered-forms](fn-unlowered-forms.md)
