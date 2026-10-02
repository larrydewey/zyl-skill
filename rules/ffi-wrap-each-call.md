# ffi-wrap-each-call

> Put each foreign function's `extern` next to one small, named Zyl wrapper, so its signature, timeout, pinning and pointer conversion live in one place.

## Why It Matters

The `(extern ...)` declaration types every call, but it cannot say what the C function means: which timeout is right, which argument is an out-parameter, who frees the result, what a NULL means. Scattering raw `ffi-call`s repeats those decisions at every site, and spreads the `ffi` capability requirement across a package. A wrapper states them once and gives the rest of the program an ordinary Zyl function with a Zyl type.

## Good

```lisp
(capabilities ffi)
(extern "strlen" (String) Int)
(extern "getenv" (String) String)

(defn c-strlen ((s String)) (ffi-call "strlen" s 1000))

; getenv's NULL for an unset variable reads as a zero-length String, so
; unset and empty are one case here; the wrapper is where that is decided.
(defn c-getenv ((name String))
  (let s (ffi-call "getenv" name 1000)
    (if (= (str-length s) 0) None (Some (str-concat "" s)))))
(defn main ()
  (begin
    (print (c-strlen "hello"))                                  ; 5
    (print (match (c-getenv "ZYL_NO_SUCH_VAR") (Some _ "set") (None "unset")))   ; unset
    0))
```

## Notes

- A program with a foreign `ffi-call` links hosted over libc, so all of libc is reachable: `strlen`, `abs`, `getenv`, `puts`, `free`, `strdup`... Each still needs its extern.
- Runtime entries need no extern and are typed by `stdlib/compiler/ffi_sigs.zyl`: `zyl_now_ms`, `zyl_argc`/`zyl_arg_str`, `zyl_getenv_str`, `zyl_err_is`, `zyl_ref_new`/`zyl_ref_get`/`zyl_ref_set`, ...
- Every `ffi-call`/`ffi-pin`/`ffi-unpin` of a foreign symbol needs the `ffi` capability: in `zyl.pkg`, or `(capabilities ffi)` at the top of a lone file ([pkg-lone-file-capabilities](pkg-lone-file-capabilities.md)). A `zyl_*` runtime entry needs no grant.
- An `extern` may carry a default timeout, `(extern "getpid" () Int :timeout 1000)`, so its calls omit the last argument ([ffi-extern-required](ffi-extern-required.md)).

## See Also

- [ffi-extern-required](ffi-extern-required.md)
- [ffi-timeout-always-last](ffi-timeout-always-last.md)
- [ffi-linking](ffi-linking.md)
