# secret-branchless-masks

> Make secret-dependent decisions with masks from `math/secret/secret` (`ct-select`, `ct-eq`, `ct-is-zero`, `ct-mask`), never with `if`.

## Why It Matters

Both operands of `ct-select` are always evaluated and read; only the mask decides which bits survive, so timing and memory access are independent of the secret. The primitives take `(Secret Int)` operands and return an `Int` 0/1 (or a mask), never a `Bool`, so feeding a result to `if` is rejected twice over: the secret checker reports `E_CT_VIOLATION` first, and the condition, an `Int`, is also a type error (conditions are `Bool`).

## Vocabulary

```lisp
(capabilities secret)
(use math/words)
(use math/secret/secret)
(defn main ()
  (let a (w-from-string "tag")
    (let b (w-from-string "tag")
      (begin
        (print (declassify (ct-mask 1)))          ; -1: all ones if c = 1, zero if c = 0
        (print (declassify (ct-is-zero 0)))       ; 1, branchless
        (print (declassify (ct-is-nonzero 5)))    ; 1
        (print (declassify (ct-select 1 10 20)))  ; 10: a if c = 1 else b
        (print (declassify (ct-eq 3 3)))          ; 1
        (print (declassify (ct-ne 3 3)))          ; 0
        (print (declassify (ct-eq-words a b 3)))  ; 1: full-length compare of two Words arrays
        (print (if (ct-eq-bool 3 3) "same" "differ"))                 ; same: the declassified verdict, a Bool
        (print (if (ct-eq-words-bool a b 3) "same" "differ"))         ; same
        0))))
```

## Bad

```lisp
(defn pick ((secret-flag Secret) a b)
  (if (= secret-flag 1) a b))                ; E_CT_VIOLATION: secret-dependent branch
```

## Good

```lisp
(capabilities secret)
(use math/secret/secret)
(defn pick ((secret-flag Secret) (a Secret) (b Secret))
  (ct-select secret-flag a b))
(defn main () (begin (print (declassify (pick 0 7 9))) 0))   ; 9
```

## The core trick

```lisp
(defn my-ct-is-zero ((x Secret))
  (- 1 (bit-and (shr (bit-or x (wrapping- 0 x)) 63) 1)))   ; wrapping-: -INT_MIN must not trap
```

`(bit-or x (wrapping- 0 x))` has its top bit set for every non-zero `x` (including the most negative integer, whose negation is itself), the logical `shr 63` isolates it, the subtraction inverts. Note `shr`, not `ashr`, and `wrapping-`: under the default checked arithmetic `(- 0 x)` would stop the program with `E_OVERFLOW` for the most negative `Int` (the library's own `ct-is-zero` is written the same way).

## Notes

- `math/secret/secret` needs the `secret` capability: a lone file declares `(capabilities secret)` ([pkg-lone-file-capabilities](pkg-lone-file-capabilities.md)).

## See Also

- [secret-ct-eq-words-for-bytes](secret-ct-eq-words-for-bytes.md)
- [bits-shr-vs-ashr](bits-shr-vs-ashr.md)
