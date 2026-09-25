# syn-int-literal-range

> Write any constant at or above 2^63 as its negative two's-complement `Int`; never write an unsigned-sized literal.

## Why It Matters

`Int` is signed 64-bit. A literal outside the Int64 range is **not rejected**: the conversion returns 0 and the program compiles with `0` in its place. This hits every hash, cipher and mixing constant written in its natural unsigned hex form. Decimal and hex are equally affected, and the type checker cannot help: the `0` is a well-typed Int. The boundary is exact: `-9223372036854775808` and `0x7FFFFFFFFFFFFFFF` are fine, `9223372036854775808` is 0.

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
- Literal forms: decimal, `0x`/`0o`/`0b` prefixes (either case), optional leading `-`. No suffixes and no digit separators: `1_000` reads as `1` followed by the identifier `_000` (`E_UNBOUND_VARIABLE`); `0xZZ` reads as a number `0x` then the identifier `ZZ`.
- Floats need a leading digit and a `.` or exponent: `1.5`, `1.`, `1e3`. `.5` is `E_INVALID_CHAR`. No `inf`/`nan` literals. `1.2.3` is not rejected by the lexer or the type checker and fails in the assembler ("junk at end of line").
- An Int literal where a Float is expected is a type error, not a conversion: write `2.0`, not `2` ([fn-int-float-separation](fn-int-float-separation.md)).
- Unsigned comparison of such values needs `math/bits` (`u64-lt` etc.): `<` is a signed compare.

## See Also

- [bits-shr-vs-ashr](bits-shr-vs-ashr.md) - bit patterns vs numbers
- [fn-integer-arith-unchecked](fn-integer-arith-unchecked.md) - overflow wraps
