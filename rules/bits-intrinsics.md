# bits-intrinsics

> Use the deterministic bit intrinsics (`bit-popcount`, `bit-clz`, `bit-ctz`, `bit-bswap`, `bit-rotl`, `bit-rotr`, `mul-hi`, `mul-hi-u`, `crc32c`, `crc32c-u8`, and the `...32` forms) and `stdlib/simd` lane vectors instead of loops or inline assembly; there is no inline assembly.

## Why It Matters

Spec §21.13: each intrinsic is defined for every input and gives the same result on every CPU, so it keeps the determinism contract that raw assembly (`rdtsc`, `cpuid`, feature-dependent paths) would break. Each compiles to inline machine code or one runtime step. They are builtins: no `use`, no `extern`, and a user program cannot name the `%` primitives behind them (`E_FFI_RESTRICTED`).

| Form | Result |
|---|---|
| `(bit-popcount x)` | set bits, 0..64 |
| `(bit-clz x)`, `(bit-ctz x)` | leading / trailing zeros; 64 for 0 |
| `(bit-bswap x)` | the 8 bytes reversed |
| `(bit-rotl x n)`, `(bit-rotr x n)` | rotate by n mod 64 (-1 is 63) |
| `(mul-hi a b)`, `(mul-hi-u a b)` | high 64 bits of the signed / unsigned 128-bit product |
| `(crc32c crc x)` | one CRC-32C step over x's 8 bytes, low byte first; state is crc's low 32 bits; no pre- or post-inversion; SSE4.2 or a table, same result |
| `(crc32c-u8 crc b)` | the same over one byte |
| `bit-popcount32`, `bit-clz32`, `bit-ctz32` (32 for 0), `bit-bswap32`, `bit-rotl32`, `bit-rotr32` | on the low 32 bits, result in 0..2^32-1 |

`stdlib/simd` (`(use simd/simd)`) gives `I64x2`, `I32x4` and `U8x16`: 128-bit vectors as immutable values, lane-wise `-add`, `-sub`, `-and`, `-or`, `-xor`, `-eq` (all-ones lanes), `-min`, `-max`, `-hsum`, `-get`/`-set`, `-splat`, and `u8x16-movemask`. They are portable SWAR over two words, not SSE, so results never depend on the CPU; a lane index out of range is `E_INDEX_OUT_OF_BOUNDS`.

## Bad

```lisp
(defn popcount (x acc)                       ; a loop the hardware does in one instruction
  (if (= x 0) acc (popcount (shr x 1) (+ acc (bit-and x 1)))))
```

## Good

```lisp
(defn main ()
  (begin
    (print (bit-popcount 255))               ; 8
    (print (bit-ctz 8))                      ; 3
    (print (bit-rotl 1 65))                  ; 2
    (print (mul-hi-u -1 2))                  ; 1
    0))
```

## Notes

- To checksum a string of bytes, fold `crc32c-u8` over them and invert at the ends yourself if your format wants the conventional CRC-32C (start at `0xFFFFFFFF`, xor at the end).
- A definition of the same name in a module shadows the builtin there.
- Tests: `tests/regression/intrinsics.zyl`, `simd.zyl`.

## See Also

- [bits-defined-shift-counts](bits-defined-shift-counts.md)
- [bits-shr-vs-ashr](bits-shr-vs-ashr.md)
