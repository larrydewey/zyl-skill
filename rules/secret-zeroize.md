# secret-zeroize

> Frames that held secrets are zeroed on return; erase heap key material explicitly with `(zeroize base n)`, `(zeroize-bytes base n)` or `(k.wipe)`.

## Why It Matters

Keys left in memory end up in core dumps. **Stack frames are erased automatically:** a function with a `Secret` parameter (or a parameter of a type implementing the `Secret` trait), a secret-returning function, and any function that binds a secret-derived `let` zeroes its whole frame when it returns (`rep stosq`, result kept in a register) and makes no tail calls. **Heap contents are not**: erase them yourself with `zeroize`/`zeroize-bytes` (both write through a volatile pointer in the runtime, so the C compiler cannot delete the stores) or with `(k.wipe)` for a type implementing `Secret`. Nothing is wiped at scope exit — Zyl does not track moves, so wiping a value stored elsewhere would destroy live data.

A function that takes a `Secret`, returns a public result, is not a declassifying helper, and never calls `zeroize`/`zeroize-bytes` gets `E_ZEROIZE_MISSING` — a **warning** on stderr despite the `E_` prefix. Functions returning a secret are not warned (the secret is still live in the caller).

## Bad

```lisp
(defn use-key (arena (key Secret) n)
  (compute arena key n))     ; frame is wiped, but the n words at `key` stay on the heap
;; E_ZEROIZE_MISSING: warning: ... consumes a Secret parameter into a public result ...
```

## Good

```lisp
(use math/secret/secret)
(defn derive-and-wipe (arena (key Secret) n)
  (let out (compute arena key n)
    (begin
      (zeroize key n)        ; n words at `key`
      out)))

(defstruct KeyBuf (base Int) (n Int))                     ; n words of key at `base`
(impl Secret KeyBuf (defn wipe (self) (zeroize self.base self.n)))
;; ... (k.wipe) when done with k
```

## Notes

- A trait call reached through a function value, or a `try` that unwinds past a function, skips that function's frame wipe.
- The frame wipe only covers the function's own frame: copies passed to unannotated helpers are not wiped ([secret-unannotated-helpers-launder](secret-unannotated-helpers-launder.md)).

## See Also

- [secret-annotate-params](secret-annotate-params.md)
