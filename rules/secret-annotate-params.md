# secret-annotate-params

> Mark key material with a `Secret` parameter annotation — `(k Secret)` or `(k (Secret Int))` — at every function that handles it.

## Why It Matters

`Secret` is a capability on the **binding**: it tracks where a value may go, not its shape. `secret_check.zyl` propagates taint forward through `let`, calls, arithmetic, constructors and byte loads. A function whose body is tainted under its own `Secret` parameters is **secret-returning**: its result is tainted at every call site, even with public literal arguments (computed by a fixpoint capped at eight rounds). Every function in `math/secret/secret` is secret-returning.

## Bad

```lisp
(use math/secret/secret)
(print (ct-select 1 10 20))            ; E_SECRET_DEBUG: result is secret
```

## Good

```lisp
(use math/secret/secret)
(defn check ((k (Secret Int)))
  (if (declassify (ct-eq k 5)) 1 0))   ; explicit declassification
(print (declassify (ct-select 1 10 20)))   ; 10
```

## Notes

- Programs with no `Secret` annotation are completely unaffected.
- Inside a package, calling `stdlib/math/secret` (or using the `Secret` type) requires the `secret` capability. A bare annotation works in any program.
- Secret diagnostics print as unlocated `PANIC:` lines with the qualified function name, e.g. `in local/main@0::app::leak`.
- Annotations currently stop at `math/secret/secret`: AEAD, KDF, signature and bignum entry points are not yet under the checker.

## See Also

- [secret-five-prohibitions](secret-five-prohibitions.md)
- [secret-unannotated-helpers-launder](secret-unannotated-helpers-launder.md)
