# secret-declassify-explicitly

> Make a secret-derived value public only through `declassify`, `ct-eq-bool` or `ct-eq-words-bool`, with a comment saying why it is safe.

## Why It Matters

Some values must become public: an AEAD acts on its tag verdict, a signature is published, a ciphertext is sent. `declassify` is the identity function whose whole job is to be a greppable statement of intent. `ct-eq-bool`/`ct-eq-words-bool` declassify by construction, reducing a comparison to one public bit. Every declassification in `stdlib/math` is such a call, commented.

## Good

```lisp
(use math/secret/secret)
;; The AEAD verdict is the only thing revealed, and the caller acts on it anyway.
(if (ct-eq-words-bool tag computed 16) (Some pt) None)

;; Public by protocol: a signature is published.
(declassify signature-word)
```

## Legitimate declassification sites in stdlib

AEAD/MAC/signature verdicts (via `ct-eq-words-bool`), Miller–Rabin rejection, RFC 6979 retry loop, public-key decompression, X25519 all-zero output check (RFC 7748 §6.1), RSA-OAEP combined extraction bit.

## See Also

- [secret-zeroize](secret-zeroize.md)
