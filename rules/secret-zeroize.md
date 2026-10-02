# secret-zeroize

> Frames that held secrets are zeroed on return; erase heap key material explicitly with `(zeroize words n)`, `(zeroize-bytes addr n)` or `(k.wipe)`.

## Why It Matters

Keys left in memory end up in core dumps. **Stack frames are erased automatically:** a function with a `Secret` parameter (or a parameter of a type implementing the `Secret` trait), a secret-returning function, and any function that binds a secret-derived `let` zeroes its whole frame when it returns (`rep stosq`, result kept in a register) and makes no tail calls. **Heap contents are not**: erase them yourself with `zeroize` (a `Words` array, word by word through the runtime's bounds-checked `zyl_words_set`, which no C compiler can delete), `zeroize-bytes` (a raw address, written through a volatile pointer) or `(k.wipe)` for a type implementing `Secret`. Nothing is wiped at scope exit — Zyl does not track moves, so wiping a value stored elsewhere would destroy live data.

A function that takes a `Secret`, returns a public result, is not a declassifying helper, and never calls `zeroize`/`zeroize-bytes` gets `E_ZEROIZE_MISSING` — a **warning** on stderr despite the `E_` prefix. Functions returning a secret are not warned (the secret is still live in the caller).

## Bad

```lisp
(capabilities secret)
(use math/words)
(use math/secret/secret)
(defn compute ((key (Secret Words))) (w-get key 0))
(defn use-key ((key (Secret Words)))
  (declassify (compute key)))  ; frame is wiped, but the words at `key` stay on the heap
;; warning[E_ZEROIZE_MISSING]: `use-key` consumes a Secret parameter into a public result without zeroize
```

## Good

```lisp
(capabilities secret)
(use math/words)
(use math/secret/secret)
(defn compute ((key (Secret Words))) (w-get key 0))
(defn derive-and-wipe ((key (Secret Words)) (n Int))
  (let out (declassify (compute key))
    (begin
      (zeroize key n)        ; words [0, n) of `key`; returns n * 8
      out)))

(defstruct KeyBuf (base Words) (n Int))                   ; n words of key material
(impl Secret KeyBuf (defn wipe (self) (zeroize self.base self.n)))

(defn main ()
  (let k (make-KeyBuf (w-from-hex "2a2b") 2)
    (begin
      (print (derive-and-wipe (w-from-hex "0102") 2))     ; 1
      (k.wipe)                                            ; its result is secret too: do not print it
      0)))
```

## Notes

- `zeroize` takes a `Words` handle (`math/words`), not a raw address; passing an `Int` is `E_TYPE_MISMATCH`. `zeroize-bytes` takes a raw address from C (an `Int`).
- A trait call reached through a function value, or a `try` that unwinds past a function, skips that function's frame wipe.
- The frame wipe only covers the function's own frame: passing a secret to a helper whose parameter is not annotated `Secret` is now `E_SECRET_UNANNOTATED`, so every function that receives a secret is itself wiped ([secret-unannotated-param-fails](secret-unannotated-param-fails.md)).

## See Also

- [secret-annotate-params](secret-annotate-params.md)
