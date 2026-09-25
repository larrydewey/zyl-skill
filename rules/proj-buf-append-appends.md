# proj-buf-append-appends

> Use `buf-append` only on a fresh `StrBuf` from `buf-new` or one you intend to extend: it appends after what the buffer already holds, it does not copy.

## Why It Matters

`(buf-append dst src)` writes `src` at the end of the text already in `dst` and returns `dst`. The buffer is a typed `StrBuf` (`allocator/allocator`): `(buf-new arena n)` makes a zeroed one of `n` bytes, `buf-append` extends it in place, `buf-str` reads it as a `String`. Reusing a non-empty buffer accumulates old content, which is correct for output buffers (codegen emits this way) and wrong as a "copy". The opposite bug (using a copy where an append was needed) truncates output to the last line. Appending past the capacity panics with `E_INDEX_OUT_OF_BOUNDS: string buffer full`.

## Bad

```lisp
(use allocator/allocator)
(defn render ((buf StrBuf) (s String)) (buf-str (buf-append buf s)))
(let scratch (buf-new arena 64)
  (begin
    (print-string (render scratch "a"))      ; a
    (print-string (render scratch "b"))))    ; ab -- the buffer still held "a"

(buf-append (arena-alloc arena 256) "hello") ; E_TYPE_MISMATCH: an arena address is a Ptr, not a StrBuf
```

## Good

```lisp
(use allocator/allocator)
(let buf (buf-new arena 1024)
  (begin
    (buf-append buf "line 1\n")
    (buf-append buf "line 2\n")       ; accumulates: intended
    (buf-str buf)))                   ; the text as a String

(str-intern arena s)                  ; a fresh copy of s in the arena
```

## Notes

- For output of unknown size, `io/io`'s `StringBuffer` grows itself (it is written into one growable buffer, as the language server's JSON writer is); a `StrBuf` has a fixed capacity.
- Building a long string with repeated `str-concat` copies everything each time and is quadratic: the language server's JSON codec did that and peaked at 12 GB on a 158 KB file before it moved to one growable buffer.

## See Also

- [own-heap-never-freed](own-heap-never-freed.md)
- [cg-emission-appends](cg-emission-appends.md)
