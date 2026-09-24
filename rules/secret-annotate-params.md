# secret-annotate-params

> Mark key material with a `Secret` parameter annotation — `(k Secret)` or `(k (Secret Int))` — at every function that handles it.

## Why It Matters

`Secret` is a capability on the **binding** (a parameter, a struct/ADT field, or every value of a type implementing the `Secret` trait): it tracks where a value may go, not its shape. `secret_check.zyl` propagates taint forward through `let`, calls, arithmetic, constructors and byte loads. A function whose body is tainted under its own `Secret` parameters is **secret-returning**: its result is tainted at every call site, even with public literal arguments (computed by a fixpoint capped at eight rounds). Every function in `math/secret/secret` is secret-returning.

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

- **Secret fields:** a field declared `Secret` or `(Secret Int)` is secret when read — `(struct-get l "pw")`, `l.pw`, or a `match` binder in that position. Storing a secret in such a field does not taint the record, so the record itself can be printed (derived `Show` prints the field as `<secret>`).
- **Secret types:** `(impl Secret Key (defn wipe (self) ...))` (prelude trait `(trait Secret (wipe (self) Int))`) makes `Key` key material everywhere: its constructors produce secrets, a parameter or field of type `Key` is secret without an annotation, it prints as `<secret>`, and `(k.wipe)` erases it. A `Show` impl or derive for it is `E_IMPL_FORBIDDEN` (prelude `(impl-not Show Secret)`).
- Programs with no `Secret` annotation are completely unaffected.
- Inside a package, calling `stdlib/math/secret` (or using the `Secret` type) requires the `secret` capability. A bare annotation works in any program.
- Secret diagnostics print as unlocated `PANIC:` lines with the qualified function name, e.g. `in local/main@0::app::leak`.
- Annotations currently stop at `math/secret/secret`: AEAD, KDF, signature and bignum entry points are not yet under the checker.
- `set!` of a secret into an existing `let-mut` variable is not tracked: the variable stays public.

## See Also

- [secret-five-prohibitions](secret-five-prohibitions.md)
- [secret-unannotated-helpers-launder](secret-unannotated-helpers-launder.md)
