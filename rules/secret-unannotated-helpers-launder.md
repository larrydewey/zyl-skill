# secret-unannotated-helpers-launder

> Annotate every helper a secret flows through; an unannotated helper silently launders taint.

## Why It Matters

Taint enters a callee only through **annotated** parameters. Pass a secret to a helper whose parameter is plain, and inside that helper the value is public: the helper can branch on it, index with it or print it without any diagnostic. The checker is syntactic and fails in the permissive direction: a shape it does not walk misses a diagnostic, it never rejects valid code.

## Bad

```lisp
(defn is-weak (k) (if (< k 1000) 1 0))       ; unannotated: branches on the key freely
(defn check ((k Secret)) (is-weak k))
```

## Good

```lisp
(defn is-weak ((k Secret)) ...)              ; now the branch is E_CT_VIOLATION
```

## Notes

- The checker is not a proof. `verify/timing.py` (dudect-style Welch t-test with a deliberately leaky positive control) measures gross leakage; run it via `./run_regression_tests.sh --full --no-boot --filter timing`.
- `print` of a secret is rejected, not redacted.

## See Also

- [secret-annotate-params](secret-annotate-params.md)
