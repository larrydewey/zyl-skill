# secret-unannotated-param-fails

> Annotate every parameter a secret flows into: passing a `Secret` value to a top-level function whose parameter is not marked `Secret` is `E_SECRET_UNANNOTATED`. Mark the parameter, or pass `(declassify ...)` when the value is genuinely public.

## Why It Matters

Taint enters a callee only through annotated parameters. Until 2026-10-02 an unannotated helper silently laundered a secret: inside it the value was public, so it could branch on it, index with it or print it without a diagnostic. The checker now fails closed across a call. A tainted argument to a top-level `defn` must meet a parameter annotated `Secret`, `(Secret Int)` or `(Secret Words)`; otherwise the call is refused at the argument, naming the parameter to annotate:

```text
error[E_SECRET_UNANNOTATED]: `k` is Secret, and `is-weak` takes it as `k`, not marked Secret
  = note: unmarked, `is-weak` could branch on it or print it, and nothing would check
  = help: write `(k Secret)` in the parameters of `is-weak`, or pass `(declassify ...)` if the value is public
```

Once the parameter is marked, the helper's body is checked like any other secret code, so a branch on the value there becomes `E_CT_VIOLATION`.

## Bad

```lisp
(defn is-weak (k) (if (< k 1000) 1 0))
(defn check ((k Secret)) (is-weak k))   ; E_SECRET_UNANNOTATED
```

```lisp
(defn is-weak ((k Secret)) (if (< k 1000) 1 0))   ; E_CT_VIOLATION: branch on a secret
(defn check ((k Secret)) (is-weak k))
```

## Good

```lisp
(capabilities secret)
(use math/secret/secret)
(defn weak-mask ((k (Secret Int))) (ct-mask (ct-is-zero k)))
(defn length-of ((n Int)) (* n 8))
(defn check ((k (Secret Int)) (n Int))
  (begin
    (print (length-of n))                    ; 16: n is public
    (print (declassify (weak-mask k)))       ; -1: a declassified verdict
    (print (length-of (declassify k)))       ; 0: declassified before the call
    0))
(defn main () (check 0 2))
```

## Notes

- Exempt callees: the declassifiers (`declassify`, `ct-eq-bool`, `ct-eq-words-bool`) and the standard library's own erasers and index-guarded memory primitives.
- Not covered yet: calls through a function value and trait method calls. A secret passed that way still enters the callee unchecked, so do not route key material through closures or traits.
- The checker is syntactic and is not a proof. `verify/timing.py` (a dudect-style Welch t-test with a deliberately leaky positive control) measures gross leakage: `./run_regression_tests.sh --full --no-boot --filter timing`.
- `print` of a secret is `E_SECRET_DEBUG`; only a secret inside a `Secret` field of a printed record is redacted (`<secret>`).
- A `let-mut` that is ever `set!` to a secret is secret for its whole scope.
- Calling `math/secret/secret` from a lone file needs `(capabilities secret)` ([pkg-lone-file-capabilities](pkg-lone-file-capabilities.md)); a bare `Secret` annotation needs nothing.

## See Also

- [secret-annotate-params](secret-annotate-params.md)
- [secret-declassify-explicitly](secret-declassify-explicitly.md)
- [secret-five-prohibitions](secret-five-prohibitions.md)
