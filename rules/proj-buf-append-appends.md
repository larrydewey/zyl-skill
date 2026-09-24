# proj-buf-append-appends

> Use `buf-append` only on a fresh zeroed buffer or one you intend to extend: it appends at `strlen(dst)`, it does not copy.

## Why It Matters

`(buf-append dst src)` writes `src` at the end of the NUL-terminated string already in `dst`. Reusing a non-empty buffer accumulates old content; using a non-zeroed buffer appends after whatever garbage precedes the first NUL. This is correct for output buffers (codegen emits this way) and wrong as a "copy". The opposite bug (using a copy where an append was needed) truncates output to the last line.

## Bad

```lisp
(let buf (arena-alloc arena 256)       ; not zeroed: appends after garbage
  (buf-append buf "hello"))
```

## Good

```lisp
(use allocator/allocator)
(let buf (arena-alloc-zeroed arena 1024)
  (begin
    (buf-append buf "line 1\n")
    (buf-append buf "line 2\n")       ; accumulates: intended
    (str-concat "" buf)))              ; copy out as a Zyl string

(str-intern arena s)                   ; a fresh copy of s in the arena
```

## See Also

- [own-heap-never-freed](own-heap-never-freed.md)
- [cg-emission-appends](cg-emission-appends.md)
