# syn-int-literal-range

> Write any constant at or above 2^63 as its negative two's-complement `Int`: an unsigned-sized literal is `E_INTEGER_OVERFLOW`. Multiply or add such constants with the `wrapping` operators, since checked arithmetic traps on the wrap a hash relies on.

## Why It Matters

`Int` is signed 64-bit. A literal outside the Int64 range is a located compile error, `E_INTEGER_OVERFLOW` ("integer literal 0xFF51AFD7ED558CCD does not fit an Int (64-bit signed)"); it used to compile silently as `0`. This hits every hash, cipher and mixing constant written in its natural unsigned hex form. The boundary is exact: `-9223372036854775808` and `0x7FFFFFFFFFFFFFFF` are fine, `9223372036854775808` is an error.

Integer arithmetic is checked by default, so the multiply in a mixing function would stop the program with `E_OVERFLOW` the first time it wraps. Spell the wrap you want with `wrapping*`, `wrapping+` and `wrapping-`, or declare `(numeric wrapping)` for the whole file ([fn-numeric-policy](fn-numeric-policy.md)).

## Bad

Each definition is its own error.

```lisp
(defn mix (x)
  (* x 0xFF51AFD7ED558CCD))        ; E_INTEGER_OVERFLOW: does not fit an Int

(defn big () 18397679294719823053) ; E_INTEGER_OVERFLOW
```

## Good

```lisp
(defn mix (x)
  (wrapping* x -49064778989728563)) ; 0xFF51AFD7ED558CCD - 2^64: the same bits

(defn main ()
  (begin
    (print (mix 3))                 ; -147194336969185689
    0))
```

## Notes

- `wrapping+`, `wrapping-`, `wrapping*`, `bit-xor`, `bit-and`, `bit-or` and the shifts do not care about sign, so the negative spelling produces the identical bit pattern. `stdlib/math/hash/sha512.zyl` writes all its round constants this way.
- Literal forms: decimal, `0x`/`0o`/`0b` prefixes (either case), optional leading `-`. No suffixes and no digit separators: `1_000` reads as `1` followed by the identifier `_000` (`E_UNBOUND_VARIABLE`); `0xZZ` reads as a number `0x` then the identifier `ZZ`.
- Floats need a leading digit and a `.` or exponent: `1.5`, `1.`, `1e3`. `.5` is `E_INVALID_CHAR`. No `inf`/`nan` literals. `1.2.3` is not rejected: it compiles and prints as `1.200000`.
- An Int literal where a Float is expected is a type error, not a conversion: write `2.0`, not `2` ([fn-int-float-separation](fn-int-float-separation.md)).
- Unsigned comparison of such values needs `math/bits` (`u64-lt` etc.): `<` is a signed compare.

## See Also

- [bits-shr-vs-ashr](bits-shr-vs-ashr.md) - bit patterns vs numbers
- [fn-numeric-policy](fn-numeric-policy.md) - checked, wrapping and saturating arithmetic
