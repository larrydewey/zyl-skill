# test-unimplemented-features

> Don't use `test-suite`, `setup`/`teardown`, `test-property`, `test-compile`, `assert-fail`, keyword options or the `run-tests-*` helpers: they compile and do nothing (or silently drop tests).

## Why It Matters

The worst one is `test-suite`: the tests nested inside it are **silently dropped** — neither registered nor run — so a suite looks green while testing nothing.

| Feature | Behavior |
|---|---|
| `(test-suite "name" (test ...) ...)` | nested tests dropped |
| `setup` / `teardown` | accepted at top level, never run |
| `(test-property "n" gen prop)`, `gen-int`/`gen-bool`/`gen-string`/`gen-float`, `property-int` & friends | compiled, never run |
| `test-compile` | no effect |
| `assert-fail` | always passes |
| `assert` | works: a failing `assert` fails the test (it panics; the message is not shown) |
| `:parallel`, `:filter`, `:verbose` keywords on `test`/`run-tests` | ignored |
| `run-tests-filtered`, `run-tests-parallel`, `run-tests-with-timeout`, `test-count` (`testing/testing`) | placeholders that call `error` |

## Bad

```lisp
(test-suite "math"
  (test "add" (assert-equal (+ 1 2) 3)))   ; never runs
(run-tests)                                ; test result: 0 passed ...
```

## Good

```lisp
(test "math-add" (assert-equal (+ 1 2) 3))
(run-tests)
```

## See Also

- [test-toplevel-forms-and-run-tests](test-toplevel-forms-and-run-tests.md)
- [fn-unlowered-forms](fn-unlowered-forms.md)
