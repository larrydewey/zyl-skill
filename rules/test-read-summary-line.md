# test-read-summary-line

> Judge a test run by its output (`FAIL` lines and the `test result:` summary). The exit status now correctly reflects pass/fail (fixed 2026-09-29).

## Why It Matters

Previously, `zyl_run_tests` returned 1 when any test failed, but that value did not become the process exit status: a test binary exited 0 with failures. This was fixed by making the implicit `main` return the last statement's value (the `zyl_run_tests` result). Zyl's own `run_regression_tests.sh` still greps for `FAIL` as a defense in depth. Inside a test, assertion messages are not printed; failures report only `FAIL` (outside a test, `assert`/`assert-true` panic with a string-literal message).

## Good

```bash
./my-tests | tee out.txt
grep -q 'FAIL' out.txt && exit 1
grep 'test result:' out.txt
```

## Notes

- Name tests descriptively and keep one behavior per test, since the name is the only diagnostic.
- Limits: at most 256 tests per program; names truncated to 127 characters; no command-line filter; tests run sequentially in one process. The only isolation is region unwinding: a failing test's frame regions are released, but heap values, `def` globals and files persist into the next test.

## See Also

- [test-toplevel-forms-and-run-tests](test-toplevel-forms-and-run-tests.md)
