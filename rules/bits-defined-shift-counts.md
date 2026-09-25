# bits-defined-shift-counts

> Rely on Zyl's defined shift semantics (logical shifts by ≥64 give 0; `ashr` saturates), not on x86's mod-64 masking.

## Why It Matters

x86 masks shift counts mod 64 (`shl 1 65` = 2). Zyl does not expose that: a logical shift by 64 or more yields 0, `ashr` by ≥64 yields the sign fill, and negative counts behave like huge counts. Porting C/Rust code that relies on masking changes results.

```lisp
(shl 1 64)       ; 0, not 1
(shl 1 65)       ; 0
(shr -1 64)      ; 0
(ashr -1 64)     ; -1
(ashr 1024 64)   ; 0
```

## Notes

- Cost: three branchless extra instructions per logical shift (`cmp`, `sbb`, `and` in the native backend), compare + cmov per `ashr`.
- `rotl64` built from shifts is correct for n in 1..63; at n = 0 the right shift by 64 gives 0, which happens to produce the right answer.
- Bitwise operators and shifts are **not constant-folded** (`optimization.zyl` folds only arithmetic and comparisons, ops 0-10); they are cheap anyway.

## See Also

- [bits-shr-vs-ashr](bits-shr-vs-ashr.md)
