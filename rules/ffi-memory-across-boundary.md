# ffi-memory-across-boundary

> Copy C strings into Zyl strings with `(str-concat "" ptr)` before freeing them; give C writable buffers from `alloc-malloc` or an arena.

## Why It Matters

A pointer returned by C is just an `Int` in Zyl: `print` shows the number, and if C `malloc`ed it, Zyl owns it and must `free` it. Zyl strings passed to C are Zyl-owned: C may read them during the call but must not write, free or keep them.

## Good

```lisp
(use allocator/allocator)

(defn c-owned-string ()
  (let p (ffi-call "strdup" "copied by C" 1000)
    (let s (str-concat "" p)          ; copy into Zyl memory
      (let _ (ffi-call "free" p 1000) ; then release C's allocation
        s))))

(defn reverse-string (s)
  (let buf (alloc-malloc 256)         ; or (arena-alloc-zeroed arena 256)
    (let _ (ffi-call "ml_reverse" s buf 256 1000)
      (let out (str-concat "" buf)
        (let _ (alloc-free buf)
          out)))))

(print (str-concat "HOME=" (ffi-call "getenv" "HOME" 1000)))
```

## Notes

- `getenv` returns 0 when unset; check before using as a string.
- The runtime's string helpers reject non-zero pointers below `0x1000`; an indirect call through such a value exits with `zyl: invalid callee address`. Nothing else protects Zyl memory from C: test C with ASan/UBSan.
- No callbacks: C cannot call Zyl functions. Poll a C function returning an event code instead.

## See Also

- [ffi-pin-passes-pointer](ffi-pin-passes-pointer.md)
- [own-heap-never-freed](own-heap-never-freed.md)
