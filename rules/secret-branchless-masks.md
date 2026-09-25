# secret-branchless-masks

> Make secret-dependent decisions with masks from `math/secret/secret` (`ct-select`, `ct-eq`, `ct-is-zero`, `ct-mask`), never with `if`.

## Why It Matters

Both operands of `ct-select` are always evaluated and read; only the mask decides which bits survive, so timing and memory access are independent of the secret. The primitives take `(Secret Int)` operands and return an `Int` 0/1 (or a mask), never a `Bool`, so feeding a result to `if` is rejected twice over: the secret checker reports `E_CT_VIOLATION` first, and the condition, an `Int`, is also a type error (conditions are `Bool`).

## Vocabulary

```lisp
(use math/secret/secret)
(ct-mask c)            ; all-ones if c = 1, zero if c = 0
(ct-is-zero x)         ; 1 if x = 0, branchless
(ct-is-nonzero x)
(ct-select c a b)      ; a if c = 1 else b
(ct-eq a b)  (ct-ne a b)
(ct-eq-words a b n)  (ct-ne-words a b n)   ; full-length compare of two Words arrays
(ct-eq-bool a b)  (ct-eq-words-bool a b n)   ; the declassified verdict, a Bool
```

## Bad

```lisp
(if (= secret-flag 1) a b)
```

## Good

```lisp
(ct-select secret-flag a b)
```

## The core trick

```lisp
(defn ct-is-zero ((x Secret))
  (- 1 (bit-and (shr (bit-or x (- 0 x)) 63) 1)))
```

`(bit-or x (- 0 x))` has its top bit set for every non-zero `x` (including the most negative integer), the logical `shr 63` isolates it, the subtraction inverts. Note `shr`, not `ashr`.

## See Also

- [secret-ct-eq-words-for-bytes](secret-ct-eq-words-for-bytes.md)
- [bits-shr-vs-ashr](bits-shr-vs-ashr.md)
