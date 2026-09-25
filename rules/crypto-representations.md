# crypto-representations

> In `stdlib/math`, byte strings are `Words` arrays with one byte (0..255) per word, big numbers are 24-bit limbs least-significant first, and every entry point takes an `Arena` first.

## Why It Matters

Every module follows these conventions. The handles are typed, so passing a `ByteBuf`, a String or a bare `Int` where a `Words` array or an `Arena` is expected is `E_TYPE_MISMATCH` at compile time; what the types cannot catch is a word array holding the wrong encoding (packed bytes, or a C string's words), which produces garbage. The arena argument decides scratch-space lifetime.

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
        (print-string (w-hex-bytes (sha256-bytes a msg 3) 32))   ; the same digest
        0))))
```

## `math/words`

A `Words` value is a handle: its length plus its storage, allocated in an arena. `(w-alloc arena n)` (zeroed), `(w-len w)`, `(w-get w i)`, `(w-set w i v)`, `(w-view arena w off n)` (a sub-array sharing the parent's storage), `w-fill`, `w-copy`, `(w-from-hex arena "0a0b")`, `(w-from-string arena "abc")`, `(w-hex-bytes w n)`, `w-hex-words`. Values are `Int`: storing a String (or Float) through `w-set` is `E_TYPE_MISMATCH`. Every access is bounds-checked at run time: an index outside the array panics with `E_INDEX_OUT_OF_BOUNDS` (`w-get: index 5 outside a word array of length 2`), so no code does address arithmetic.

## Notes

- 24-bit limbs let a limb product (48 bits) plus a column of accumulated products fit a signed 64-bit Int.
- Argon2 is the exception: its blocks are 128 genuine 64-bit words.
- `(use math/math)` loads the whole ~7,600-line tree; import only the modules you use to keep compile time and binary size down.
- Hash modules share a shape: `<h>-words`, `<h>-bytes`, `<h>-hex-of-string`.
- `=` on two `Words` values compares handles, not contents ([secret-ct-eq-words-for-bytes](secret-ct-eq-words-for-bytes.md)).

## See Also

- [crypto-choose-primitives](crypto-choose-primitives.md)
- [reference: stdlib](../references/stdlib.md)
