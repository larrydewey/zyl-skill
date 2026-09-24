# test-read-summary-line

> Judge a test run by its output (`FAIL` lines and the `test result:` summary), not by the exit status.

## Why It Matters

`zyl_run_tests` returns 1 when any test fails, but per Chapter 11 that value does not currently become the process exit status: a test binary can exit 0 with failures. (Chapter 29 says a `main` ending in `(run-tests)` exits with the pass/fail status; treat the exit code as unreliable either way.) Zyl's own `run_regression_tests.sh` greps for `FAIL` for exactly this reason. Inside a test, assertion messages are not printed; failures report only `FAIL` (outside a test, `assert`/`assert-true` panic with a string-literal message).

## Good

```bash
./my-tests | tee out.txt
grep -q 'FAIL' out.txt && exit 1
grep 'test result:' out.txt
```

## Notes

- Name tests descriptively and keep one behavior per test, since the name is the only diagnostic.
- Limits: at most 256 tests per program; names truncated to 127 characters; no command-line filter; tests run sequentially in one process and one heap (no isolation between tests).

## See Also

- [test-toplevel-forms-and-run-tests](test-toplevel-forms-and-run-tests.md)
