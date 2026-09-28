# test-suites-properties-compile

> Group tests in a top-level `test-suite` with `setup`/`teardown` fixtures, check properties with `test-property` over `gen-int`, `gen-bool`, `gen-string` or `gen-float`, assert that code does or does not compile with `test-compile`, and check that an expression raises with `assert-fail`. Keyword options on `run-tests` are ignored.

## Why It Matters

Spec 20.5 is implemented (2026-09-28). The forms are rewritten on the parse tree (`compiler/desugar.zyl`) into ordinary top-level `test` forms, so they register and report like any test, and a failing test prints `FAIL: <first line of its panic message>`.

| Form | Behavior |
|---|---|
| `(test-suite "s" item...)` | top level only; items are `test`, nested `test-suite`, `test-property`, `setup`, `teardown`; tests register as `s/t` (nested: `s/inner/t`) in source order |
| `(setup e...)` / `(teardown e...)` | inside a suite: run before / after each of its tests, outer suite's setup first and inner teardown first; teardown also runs when the test fails. Elsewhere `E_MALFORMED_FORM` |
| `(test-property "p" gen (fn (x ...) bool))` | one to three parameters; runs over the generator's fixed samples (edge cases such as 0, ±1, the Int limits and `""`, then a seeded sequence; no NaN or infinity); extra parameters pair each sample with the list rotated by 7 and 13; fails with ``property `p` failed for <sample>`` |
| `(test-compile e)` / `(test-compile e (:expect-error true))` | top level only; decided at compile time by running the checks and the type checker on the program with `e` as a function body; becomes a test `test-compile line N` that fails with the reason (`did not compile: error[...]` or `compiled, but an error was expected`) |
| `(assert-fail e)` / `(assert-fail e "msg")` | fails unless evaluating `e` raises; the message is a string literal |
| `(run-tests :parallel true)` | the keyword is ignored: tests run one at a time in registration order |
| a keyword on `test` | `E_MALFORMED_FORM` (a test is a name and one body form) |

## Bad

```lisp
(defn check-all ()
  (test-suite "s" (test "t" (assert-true true))))   ; E_MALFORMED_FORM: test-suite is a top-level form

(test-property "sq" gen-int (fn (x) (>= (* x x) 0)))  ; FAIL: property `sq` failed for 4294967295 (x*x wraps)
```

## Good

```lisp
(test-suite "math"
  (setup (print "setup"))
  (teardown (print "teardown"))
  (test "add" (assert-equal (+ 1 2) 3))
  (test-property "commutes" gen-int (fn (a b) (= (+ a b) (+ b a)))))

(test "raises" (assert-fail (unwrap None)))
(test-compile (+ 1 2))
(test-compile (+ 1 "a") (:expect-error true))
(run-tests)
```

## Notes

- Samples come from `core/property` (`property-samples-int` and so on), which the prelude loads; a property's counterexample is printed with `Debug`.
- The language server skips top-level `test-compile` forms: their verdict is only known at compile time.
- `testing/testing` holds only the assertions as functions (`assert-equal-values`, `assert-true-value`, `assert-false-value`, `assert-fail-call`).
- A `defn` named after a special form (`setup`, `test`) is accepted, but a call of it is the special form: never name a function after one.

## See Also

- [test-toplevel-forms-and-run-tests](test-toplevel-forms-and-run-tests.md)
- [test-read-summary-line](test-read-summary-line.md)
