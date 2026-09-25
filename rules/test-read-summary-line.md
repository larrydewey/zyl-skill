# test-read-summary-line

> Judge a test run by its output (`FAIL` lines and the `test result:` summary), not by the exit status.

## Why It Matters

`zyl_run_tests` returns 1 when any test fails, but that value does not become the process exit status: a test binary exits 0 with failures (verified 2026-09-25: `test result: 3 passed, 1 failed` and exit status 0). Zyl's own `run_regression_tests.sh` greps for `FAIL` for exactly this reason. Inside a test, assertion messages are not printed; failures report only `FAIL` (outside a test, `assert`/`assert-true` panic with a string-literal message).

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
