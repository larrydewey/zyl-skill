# secret-declassify-explicitly

> Make a secret-derived value public only through `declassify`, `ct-eq-bool` or `ct-eq-words-bool`, with a comment saying why it is safe.

## Why It Matters

Some values must become public: an AEAD acts on its tag verdict, a signature is published, a ciphertext is sent. `declassify` is the identity function whose whole job is to be a greppable statement of intent. `ct-eq-bool`/`ct-eq-words-bool` declassify by construction, reducing a comparison to one public bit. Every declassification in `stdlib/math` is such a call, commented.

## Good

```lisp
(capabilities secret)
(use math/words)
(use math/secret/secret)
(defn verdict (tag computed pt)
  ;; The AEAD verdict is the only thing revealed, and the caller acts on it anyway.
  (if (ct-eq-words-bool tag computed 16) (Some pt) None))
(defn main ()
  (begin
    (print (match (verdict (w-alloc 16) (w-alloc 16) "pt") (Some p p) (None "forged")))   ; pt
    (print (declassify 42))   ; public by protocol, like a published signature word
    0))
```

## Legitimate declassification sites in stdlib

AEAD/MAC/signature verdicts (via `ct-eq-words-bool`), Miller–Rabin rejection, RFC 6979 retry loop, public-key decompression, X25519 all-zero output check (RFC 7748 §6.1), RSA-OAEP combined extraction bit.

## Notes

- `declassify` is also the explicit escape from an `impl-not` flow rule ([trait-coherence-and-orphans](trait-coherence-and-orphans.md)) and from the redaction of `Secret` fields: there is no `Show` override for a Secret type.
- It lives in `math/secret/secret`; without that `use`, `declassify` is an unbound function.

## See Also

- [secret-zeroize](secret-zeroize.md)
