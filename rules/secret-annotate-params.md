# secret-annotate-params

> Mark key material with a `Secret` parameter annotation — `(k Secret)`, `(k (Secret Int))` or `(k (Secret Words))` — at every function that handles it.

## Why It Matters

`Secret` is a capability on the **binding** (a parameter, a struct/ADT field, or every value of a type implementing the `Secret` trait): it tracks where a value may go, not its shape. `secret_check.zyl` propagates taint forward through `let`, calls, arithmetic, constructors and byte loads. A function whose body is tainted under its own `Secret` parameters is **secret-returning**: its result is tainted at every call site, even with public literal arguments (computed by a fixpoint capped at eight rounds). The `ct-*` functions in `math/secret/secret` are secret-returning; `ct-eq-bool`, `ct-eq-words-bool` and `declassify` are recognised by name as declassifying.

The annotation is also a type: `(Secret Int)` and `(Secret Words)` fix the value's type, while a bare `Secret` leaves it to inference. The secret checker runs before the type checker, so a secret violation is reported even in a program that would also fail to type-check.

## Bad

```lisp
(capabilities secret)
(use math/secret/secret)
(defn main () (begin (print (ct-select 1 10 20)) 0))   ; E_SECRET_DEBUG: the result is secret
```

## Good

```lisp
(capabilities secret)
(use math/secret/secret)
(defn check ((k (Secret Int)))
  (if (ct-eq-bool k 5) 1 0))                     ; ct-eq-bool: a public Bool
(defn check2 ((k (Secret Int)))
  (if (= (declassify (ct-eq k 5)) 1) 1 0))       ; ct-eq is an Int 0/1: compare it
(defn main ()
  (begin
    (print (check 5))                            ; 1
    (print (check2 4))                           ; 0
    (print (declassify (ct-select 1 10 20)))     ; 10
    0))
```

## Notes

- **Secret fields:** a field declared `Secret` or `(Secret Int)` is secret when read — `(struct-get l "pw")`, `l.pw`, or a `match` binder in that position. Storing a secret in such a field does not taint the record, so the record itself can be printed (derived `Show` prints the field as `<secret>`).
- **Secret types:** `(impl Secret Key (defn wipe (self) ...))` (prelude trait `(trait Secret (wipe (self) Int))`) makes `Key` key material everywhere: its constructors produce secrets, a parameter or field of type `Key` is secret without an annotation, it prints as `<secret>`, and `(k.wipe)` erases it. A `Show` impl or derive for it is `E_IMPL_FORBIDDEN` (prelude `(impl-not Show Secret)`).
- `ct-eq`, `ct-ne`, `ct-is-zero` and friends return an `Int` 0 or 1, never a `Bool` (a Bool invites a branch): `(if (ct-eq k 5) ...)` is `E_TYPE_MISMATCH`, and so is `(if (declassify (ct-eq k 5)) ...)`.
- A function that consumes a Secret parameter into a public result without erasing it can draw the `E_ZEROIZE_MISSING` warning ([secret-zeroize](secret-zeroize.md)).
- Every function a secret is passed to must annotate that parameter too, or the call is `E_SECRET_UNANNOTATED` ([secret-unannotated-param-fails](secret-unannotated-param-fails.md)).
- Programs with no `Secret` annotation are completely unaffected.
- Calling `math/secret/secret` requires the `secret` capability, in a package's `zyl.pkg` or as `(capabilities secret)` at the top of a lone file. A bare annotation works in any program.
- The secret checker, like the capability check, does not walk top-level statements outside a `defn` or `test`; put secret code in functions.
- Secret diagnostics are located `error[CODE]` diagnostics (file:line:col, caret) naming the function: `error[E_SECRET_DEBUG]: in `f`: Secret value reaches `print` ...`.
- Annotations currently stop at `math/secret/secret`: AEAD, KDF, signature and bignum entry points are not yet under the checker.
- A `let-mut` that is ever `set!` to a secret is secret for its whole scope.

## See Also

- [secret-five-prohibitions](secret-five-prohibitions.md)
- [secret-unannotated-param-fails](secret-unannotated-param-fails.md)
