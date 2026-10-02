# ffi-extern-required

> Declare every foreign symbol with `(extern "sym" (ParamType ...) ResultType)` before the program can `ffi-call` it; call `zyl_*` runtime entries without one, and never the raw ones.

## Why It Matters

Types are sound, and `ffi-call` is where an untyped word could enter the program, so every call is typed:

1. **Runtime entries** (`zyl_*` symbols the runtime exports) are typed by the compiler's table `stdlib/compiler/ffi_sigs.zyl` (for example `zyl_now_ms` is `-> Int`, `zyl_err_is` is `String String -> Bool`, `zyl_ref_new` is `a -> (Ref a)`). A few exported entries missing from the table (`zyl_abs`, `zyl_attr_*`, `zyl_cell_*`, ...) have an untyped result.
2. **Raw runtime entries** that read memory or reinterpret a word (`zyl_cstr_of_word`, `zyl_word_load`, `zyl_ptr_cstr`, `zyl_ptr_add`, `zyl_mem_read`, `zyl_mem_write`, `zyl_call_argv`, `zyl_val_*`, `zyl_view_*`, ...; the list is `ffi-raw-p`) are `E_FFI_RESTRICTED` outside the standard library: in a user program they would be the cast the language does not have. Use the typed stdlib wrappers instead (`alloc-cstr`, `alloc-read-int`, `alloc-write-int`, `alloc-offset` in `allocator/allocator`).
3. **Foreign symbols** (anything the runtime does not export, whatever its prefix) need an `(extern ...)` declaration; without one the call is `E_CANNOT_INFER` ("ffi-call to `abs`, which has no (extern ...) declaration"). Arguments unify with the declared parameter types and the call has the declared result type.

An extern is a top-level form; it may appear after its use. It may end with `:timeout N`, a default timeout in milliseconds for every call that omits its own ([ffi-extern-required](ffi-extern-required.md)). Its parameter list must be a list, `(extern "getpid" () Int)` for none; `(extern "abs" Int Int)` is `E_MALFORMED_FORM`.

## Bad

```lisp
(capabilities ffi)
(defn main () (begin (print (ffi-call "abs" -3 1000)) 0))   ; E_CANNOT_INFER: no extern
```

```lisp
(capabilities ffi)
(defn main () (begin (print (ffi-call "zyl_cstr_of_word" 4096 1000)) 0))   ; E_FFI_RESTRICTED: use alloc-cstr
```

## Good

```lisp
(capabilities ffi)
(extern "abs" (Int) Int)
(extern "strlen" (String) Int)
(extern "getpid" () Int :timeout 1000)        ; a default timeout: calls may omit it

(defn main ()
  (begin
    (print (ffi-call "abs" -5 1000))           ; 5
    (print (ffi-call "strlen" "hello" 1000))   ; 5
    (print (> (ffi-call "getpid") 0))          ; 1: a Bool prints as 1
    (print (> (ffi-call "zyl_now_ms" 1000) 0)) ; runtime entry: no extern
    0))
```

## Notes

- Declare the C signature as it is used: `(extern "strdup" (String) Ptr)` with `(extern "free" (Ptr) Unit)` keeps the returned pointer a `Ptr`, so it cannot be mistaken for a Zyl string; `alloc-cstr` copies it out.
- A C function you name `zyl_...` is a foreign symbol: it needs an extern, and a symbol with an extern runs on the timed worker. An extern for a runtime entry (one the runtime exports, or one with a table signature such as `zyl_now_ms`) is `E_FFI_RESTRICTED`.
- A foreign `ffi-call` needs the `ffi` capability: a lone file starts with `(capabilities ffi)` ([pkg-lone-file-capabilities](pkg-lone-file-capabilities.md)).
- Extern types are restricted to what fits one integer register; see [ffi-extern-word-sized-types](ffi-extern-word-sized-types.md).

## See Also

- [ffi-extern-word-sized-types](ffi-extern-word-sized-types.md)
- [ffi-wrap-each-call](ffi-wrap-each-call.md)
- [ffi-timeout-always-last](ffi-timeout-always-last.md)
