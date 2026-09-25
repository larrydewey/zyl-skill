# test-toplevel-forms-and-run-tests

> Write tests as flat top-level `(test "name" body)` forms, end the file with `(run-tests)`, and do not define `main` in a test file.

## Why It Matters

Testing is built into the language (spec §20.5). Each top-level `test` is lowered to a function `_test_<name>` registered from a synthesized `main`; `(run-tests)` runs them in source order. Leave out `(run-tests)` and the program registers its tests and exits without running any. A file with top-level `test`/`run-tests` forms **and** an explicit `(defn main ...)` is rejected with `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` — including when the `main` comes from a `use`d module.

## Good

```lisp
(use collections/vec)
(defn factorial (n) (if (<= n 1) 1 (* n (factorial (- n 1)))))

(test "factorial" (assert-equal (factorial 5) 120))
(test "edge-case-zero" (assert-equal (factorial 0) 1))
(test "vector-push"
  (assert-equal (vec-len (vec-push (vec-create (arena-create 0) 4) 42)) 1))

(run-tests)
```

```
test: factorial ... ok
test: edge-case-zero ... ok
test: vector-push ... ok
test result: 3 passed, 0 failed, 3 total
```

## Notes

- Build and run: `zyl tests.zyl -o tests && ./tests`. No import is needed for the harness.
- A test takes exactly one body form: `(test "n" a b)` is `E_MALFORMED_FORM`. Wrap several assertions in `begin`. The first failing assertion ends that test; other tests still run.
- Assertions are typed ([test-assert-equal-semantics](test-assert-equal-semantics.md)): a type error anywhere in the file fails the whole compile, so no test runs.
- Macros expand inside `test` bodies, so tests are the way to check a macro.
- In a package, `zyl test` builds (resolving `dev-deps`) and runs the root module.

## See Also

- [test-read-summary-line](test-read-summary-line.md)
- [test-unimplemented-features](test-unimplemented-features.md)
- [test-program-library-tests-split](test-program-library-tests-split.md)
