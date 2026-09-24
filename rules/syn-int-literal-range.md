# syn-int-literal-range

> Write any constant at or above 2^63 as its negative two's-complement `Int`; never write an unsigned-sized literal.

## Why It Matters

`Int` is signed 64-bit. A literal outside the Int64 range is **not rejected**: the conversion returns 0 and the program compiles with `0` in its place. This hits every hash, cipher and mixing constant written in its natural unsigned hex form. Decimal and hex are equally affected.

## Bad

```lisp
(defn mix (x)
  (* x 0xFF51AFD7ED558CCD))        ; silently compiles as (* x 0)

(defn big () 18397679294719823053) ; also 0
```

## Good

```lisp
;; 0xFF51AFD7ED558CCD - 2^64: same bit pattern, representable as Int
(defn mix (x)
  (* x -49064778989728563))
```

## Notes

- `+`, `-`, `*`, `bit-xor`, `bit-and`, `bit-or` and the shifts do not care about sign, so the negative spelling produces the identical bit pattern. `stdlib/math/hash/sha512.zyl` writes all its round constants this way.
- Literal forms: decimal, `0x`/`0o`/`0b` prefixes (either case), optional leading `-`. No suffixes.
- Floats need a leading digit and a `.` or exponent: `1.5`, `1.`, `1e3`. `.5` is not a token. No `inf`/`nan` literals. `1.2.3` is not rejected by the lexer and fails in the assembler.
- Unsigned comparison of such values needs `math/bits` (`u64-lt` etc.): `<` is a signed compare.

## See Also

- [bits-shr-vs-ashr](bits-shr-vs-ashr.md) - bit patterns vs numbers
- [fn-integer-arith-unchecked](fn-integer-arith-unchecked.md) - overflow wraps
