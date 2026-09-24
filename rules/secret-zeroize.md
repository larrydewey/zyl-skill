# secret-zeroize

> Erase key material explicitly with `(zeroize base n)` or `(zeroize-bytes base n)` when you are done with it.

## Why It Matters

Keys left in memory end up in core dumps. Erasure is **manual**: there is no automatic zeroize at scope exit. Both functions write through a volatile pointer in the runtime so the C compiler cannot delete the stores. A function that takes a `Secret`, returns a public result, is not a declassifying helper, and never calls `zeroize`/`zeroize-bytes` gets `E_ZEROIZE_MISSING` — a **warning** on stderr despite the `E_` prefix. Functions returning a secret are not warned (the secret is still live in the caller).

## Good

```lisp
(use math/secret/secret)
(defn derive-and-wipe (arena (key Secret) n)
  (let out (compute arena key n)
    (begin
      (zeroize key n)        ; n words at `key`
      out)))
```

## See Also

- [secret-annotate-params](secret-annotate-params.md)
