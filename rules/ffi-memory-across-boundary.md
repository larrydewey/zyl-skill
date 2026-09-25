# ffi-memory-across-boundary

> Declare C-owned pointers as `Ptr`, copy their bytes into a Zyl string with `alloc-cstr` before freeing them, and give C writable buffers from `alloc-malloc` or an arena.

## Why It Matters

Memory C returns is C's: if C `malloc`ed it, Zyl must `free` it, and a Zyl `String` must not point into it after that. Declaring the result as `Ptr` in the extern keeps the two apart in the types: a `Ptr` is not a `String`, so it cannot be printed or concatenated by accident, and `alloc-cstr` (`allocator/allocator`) is the one way to read its bytes into a string. Zyl strings passed to C are Zyl-owned: C may read them during the call but must not write, free or keep them.

## Good

```lisp
(use allocator/allocator)

(extern "strdup" (String) Ptr)
(extern "free" (Ptr) Unit)
(extern "strcpy" (Ptr String) Ptr)
(extern "getenv" (String) String)

(defn c-owned-string ()
  (let p (ffi-call "strdup" "copied by C" 1000)
    (let s (str-concat "" (alloc-cstr p))    ; copy into Zyl memory
      (begin (ffi-call "free" p 1000)         ; then release C's allocation
             s))))

(defn via-buffer ((s String))
  (let buf (alloc-malloc 256)                ; a Ptr; or (arena-alloc-zeroed arena 256)
    (let _ (ffi-call "strcpy" buf s 1000)
      (let out (str-concat "" (alloc-cstr buf))
        (begin (alloc-free buf) out)))))

(defn main ()
  (begin
    (print (c-owned-string))
    (print (via-buffer "abc"))
    (print (str-concat "HOME=" (ffi-call "getenv" "HOME" 1000)))
    0))
```

## Notes

- `zyl_ptr_cstr`, `zyl_mem_read` and the other raw entries behind these helpers are `E_FFI_RESTRICTED` in user code; use `alloc-cstr`, `alloc-read-int`, `alloc-write-int` and `alloc-offset`.
- Neither `String` nor `Ptr` has a null test. A C string that may be NULL (`getenv` of an unset variable) is safest declared `String` and tested with `str-length`: the runtime's string helpers read NULL as length 0 (though `(str-eq s "")` is false for it), so unset and empty become one case.
- The runtime rejects non-zero pointers below `0x1000` in its string helpers; an indirect call through such a value exits with `zyl: invalid callee address`. Nothing else protects Zyl memory from C: test C with ASan/UBSan.
- Callbacks: a top-level function passed for an extern parameter declared `(Fn (A ...) R)` is a code pointer (a `qsort` comparator, say). It runs on the FFI worker thread, sees the caller's `actor-self`, and a panic in it not caught by a `try` inside the callback ends the process. A closure argument is rejected (`E_INVALID_CAPABILITY`).
- Foreign-call arguments are always placed in the process heap, never in a frame region, so a timed-out call that is abandoned never holds released memory.

## See Also

- [ffi-extern-required](ffi-extern-required.md)
- [ffi-pin-passes-pointer](ffi-pin-passes-pointer.md)
- [own-heap-never-freed](own-heap-never-freed.md)
