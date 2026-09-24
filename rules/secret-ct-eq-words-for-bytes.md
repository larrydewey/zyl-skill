# secret-ct-eq-words-for-bytes

> Compare MACs, tags, hashes and passwords with `ct-eq-words` / `ct-eq-words-bool`, never with `=` or a loop that stops at the first difference.

## Why It Matters

`=` on two word arrays compares **addresses**, not contents. A hand-written loop that exits early leaks the length of the matching prefix — enough to forge an authentication tag byte by byte. `ct-eq-words` accumulates differences with `bit-or` over the full length and decides at the end.

## Bad

```lisp
(defn tag-eq (a b n i)
  (if (>= i n) 1
    (if (!= (w-get a i) (w-get b i)) 0 (tag-eq a b n (+ i 1)))))   ; early exit
```

## Good

```lisp
(use math/secret/secret)
(if (ct-eq-words-bool expected-tag got-tag 16)   ; one public accept/reject bit
  (Some plaintext)
  None)
```

## See Also

- [secret-declassify-explicitly](secret-declassify-explicitly.md)
- [fn-string-equality](fn-string-equality.md)
