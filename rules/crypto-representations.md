# crypto-representations

> In `stdlib/math`, byte strings are word arrays with one byte (0..255) per 8-byte slot, big numbers are 24-bit limbs least-significant first, and every entry point takes an arena first.

## Why It Matters

Every module follows these conventions; mixing a packed buffer or a C string into them produces garbage without a type error. The arena argument decides scratch-space lifetime.

## Good

```lisp
(use allocator/allocator)
(use math/words)
(use math/hash/sha2)
(defn main ()
  (let a (arena-create 0)
    (let msg (w-from-string a "abc")
      (begin
        (print-string (sha256-hex-of-string a "abc"))
        (print-string (w-hex-bytes (sha256-bytes a msg 3) 32))
        0))))
```

## `math/words`

`(w-alloc arena n)`, `(w-get base i)`, `(w-set base i v)`, `w-fill`, `w-copy`, `(w-from-hex arena "0a0b")`, `(w-from-string arena "abc")`, `(w-hex-bytes base n)`, `w-hex-words`.

## Notes

- 24-bit limbs let a limb product (48 bits) plus a column of accumulated products fit a signed 64-bit Int.
- Argon2 is the exception: its blocks are 128 genuine 64-bit words.
- `(use math/math)` loads the whole ~7,600-line tree; import only the modules you use to keep compile time and binary size down.
- Hash modules share a shape: `<h>-words`, `<h>-bytes`, `<h>-hex-of-string`.

## See Also

- [crypto-choose-primitives](crypto-choose-primitives.md)
- [reference: stdlib](../references/stdlib.md)
