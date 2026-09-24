# ffi-timeout-always-last

> Always end `ffi-call` with a timeout literal in milliseconds: `(ffi-call "sym" args... 1000)`.

## Why It Matters

ICNF lowering (`ic-ffi`) **drops the last argument unconditionally** as the timeout, without checking it is there. Forget it and your real last argument is silently discarded: C receives one argument too few and reads garbage from the register. The timeout itself is **not enforced** at run time (`E_FFI_TIMEOUT` never raised; `(ffi-call "sleep" 3 100)` sleeps three seconds).

## Bad

```lisp
(ffi-call "abs" -5)              ; abs called with whatever rdi held
(ffi-call "zyl_cstr_len" s)      ; s silently dropped
```

## Good

```lisp
(ffi-call "abs" -5 1000)                       ; 5
(ffi-call "strlen" "hello" 1000)               ; 5
(ffi-call "zyl_actor_wait_all" 1000)           ; zero real args: timeout only
```

## Notes

- The symbol must be a string literal; bytes outside `[A-Za-z0-9_]` are replaced with `_` (no assembly injection).
- `(ffi-call)` fails at link time with ``undefined reference to `_'``.
- The compiler's own source passes `0` or `1000` by convention. Write the timeout you intend: it documents the call and stays correct when enforcement lands.
- A blocking C function hangs the calling thread; bound blocking work on the C side.

## See Also

- [ffi-int64-only-no-floats](ffi-int64-only-no-floats.md)
- [ffi-wrap-each-call](ffi-wrap-each-call.md)
