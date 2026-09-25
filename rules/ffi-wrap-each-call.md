# ffi-wrap-each-call

> Put each foreign function's `extern` next to one small, named Zyl wrapper, so its signature, timeout, pinning and pointer conversion live in one place.

## Why It Matters

The `(extern ...)` declaration types every call, but it cannot say what the C function means: which timeout is right, which argument is an out-parameter, who frees the result, what a NULL means. Scattering raw `ffi-call`s repeats those decisions at every site, and spreads the `ffi` capability requirement across a package. A wrapper states them once and gives the rest of the program an ordinary Zyl function with a Zyl type.

## Good

```lisp
(extern "strlen" (String) Int)
(extern "getenv" (String) String)

(defn c-strlen ((s String)) (ffi-call "strlen" s 1000))

; getenv's NULL for an unset variable reads as a zero-length String, so
; unset and empty are one case here; the wrapper is where that is decided.
(defn c-getenv ((name String))
  (let s (ffi-call "getenv" name 1000)
    (if (= (str-length s) 0) None (Some (str-concat "" s)))))
```

## Notes

- libc is always linked: `strlen`, `abs`, `getenv`, `puts`, `free`, `strdup`... Each still needs its extern.
- Runtime entries need no extern and are typed by `stdlib/compiler/ffi_sigs.zyl`: `zyl_actor_wait_all`, `zyl_now_ms`, `zyl_argc`/`zyl_arg_str`, `zyl_getenv_str`, `zyl_err_is`, `zyl_ref_new`/`zyl_ref_get`/`zyl_ref_set`, ...
- In a package, every `ffi-call`/`ffi-pin`/`ffi-unpin` needs the `ffi` capability.

## See Also

- [ffi-extern-required](ffi-extern-required.md)
- [ffi-timeout-always-last](ffi-timeout-always-last.md)
- [ffi-linking](ffi-linking.md)
