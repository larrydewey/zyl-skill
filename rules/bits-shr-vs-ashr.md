# bits-shr-vs-ashr

> Use `shr` (logical, zero fill) for bit patterns and `ashr` (arithmetic, sign fill) for signed numbers.

## Why It Matters

The two right shifts differ as soon as the value is negative, which for hash and cipher words (stored as signed `Int`) is half the time. Using `ashr` on a bit pattern smears the sign bit and gives "works for small inputs" bugs.

## Bad

```lisp
(defn rotl64 (x n) (bit-or (shl x n) (ashr x (- 64 n))))   ; wrong for negative x
```

## Good

```lisp
(defn rotl64 (x n) (bit-or (shl x n) (shr x (- 64 n))))
(shr  -1 60)   ; 15
(ashr -1 60)   ; -1
(ashr -16 2)   ; -4
```

## Operators

| Op | Form | Meaning |
|---|---|---|
| `bit-and` `bit-or` `bit-xor` | `(op a b ...)` | n-ary, fold left, one instruction each |
| `bit-not` | `(bit-not a)` | exactly one argument (`E_ARITY_MISMATCH` otherwise); XOR with -1 |
| `shl` | `(shl a n)` | logical left |
| `shr` | `(shr a n)` | logical right |
| `ashr` | `(ashr a n)` | arithmetic right |

All operate on signed 64-bit `Int` words (a `Bool` or `Float` operand is `E_TYPE_MISMATCH`), are branchless and constant-time. `math/bits` has ready-made 32/64-bit rotations, unsigned compares and byte packing.

## See Also

- [bits-defined-shift-counts](bits-defined-shift-counts.md)
- [syn-int-literal-range](syn-int-literal-range.md)
