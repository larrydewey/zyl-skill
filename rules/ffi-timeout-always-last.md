# ffi-timeout-always-last

> Always end `ffi-call` with a positive timeout literal in milliseconds: `(ffi-call "sym" args... 1000)`. It is checked at compile time and enforced at run time.

## Why It Matters

`ffi-check-call` (`arity_check.zyl`, also run by ICNF lowering `ic-ffi`) requires the symbol to be a string literal (`E_FFI_SYMBOL_REQUIRED`) and the last argument to be a **positive integer literal** (`E_FFI_TIMEOUT_REQUIRED`, located, with a help line). The literal requirement is what stops a forgotten timeout from silently swallowing the real last argument. More than 16 arguments is `E_ARITY_MISMATCH`.

At run time a foreign symbol (anything not starting with `zyl_`) runs on a worker thread through the runtime's `zyl_ffi_timed`; the caller waits on `CLOCK_MONOTONIC` and, on expiry, raises the panic ``E_FFI_TIMEOUT: ffi call `usleep` exceeded its timeout of 50 ms``. It is catchable with `try`/`catch` and matchable by code with `recover` or `zyl_err_is`. A too-tight timeout therefore turns a slow but correct C call into an error.

## Bad

```lisp
(extern "abs" (Int) Int)
(extern "usleep" (Int) Int)
(defn magnitude (n) (ffi-call "abs" n))   ; E_FFI_TIMEOUT_REQUIRED: n is not a literal
(ffi-call "abs" -5)                       ; E_FFI_TIMEOUT_REQUIRED: -5 is not positive
(ffi-call "abs" -5 0)                     ; E_FFI_TIMEOUT_REQUIRED: 0 is rejected
(ffi-call "abs" 5)                        ; E_TYPE_MISMATCH: zero arguments and a 5 ms timeout; abs takes one
(ffi-call sym 3 1000)                     ; E_FFI_SYMBOL_REQUIRED
(ffi-call "usleep" 300000 50)             ; compiles; raises E_FFI_TIMEOUT at run time
```

## Good

```lisp
(ffi-call "abs" -5 1000)                       ; 5
(ffi-call "zyl_actor_wait_all" 1000)           ; zero real args: timeout only
(try (ffi-call "usleep" 300000 50)
  (catch e (if (ffi-call "zyl_err_is" e "E_FFI_TIMEOUT" 1000) -1 0)))   ; zyl_err_is gives a Bool
```

## Notes

- `(ffi-call "f" 5)` is read as **zero** arguments with a 5 ms timeout. A forgotten timeout is caught by the type checker whenever the arity differs from the extern or table signature (`E_TYPE_MISMATCH`, "wrong number of arguments"); it is silent only for a function whose signature really takes one fewer argument.
- The symbol's bytes outside `[A-Za-z0-9_]` are replaced with `_` (no assembly injection).
- Lowering: a runtime entry (`zyl_*` with no `extern`) is trusted code, called directly as `IFfi sym args`; its timeout is checked but unused. A symbol with an `extern` is foreign code whatever its prefix, so your own `zyl_...` C function is timed too. Any other symbol becomes `IFfi "zyl_ffi_timed" (ISymAddr sym, IStr sym, IConst ms, IConst argc, args...)`.
- An overrunning C function is **abandoned, not killed**: its worker finishes and frees itself, the caller gets a fresh worker next call. Nothing the abandoned call was handed is reclaimed (Pin slots are never freed individually; the exit-time arena teardown is skipped once any call has been abandoned). A C function that never returns leaks one thread.
- The worker belongs to the calling thread and is kept between calls, so thread-local C state such as `errno` stays consistent.
- Callbacks (e.g. a `qsort` comparator passed as a top-level function name) run on the worker thread; they see the caller's `actor-self`, and a panic in the callback not caught by a `try` inside it ends the process.
- Determinism: whether a timeout fires depends on foreign timing; spec §27 treats FFI results, timeouts included, as observable external input.
- The interpreter (`zyl eval`, REPL) enforces timeouts too, via `zyl_ffi_timed_argv`.
- The compiler's own source uses `1000` everywhere, including zero-argument runtime calls.

## See Also

- [ffi-extern-required](ffi-extern-required.md)
- [ffi-extern-word-sized-types](ffi-extern-word-sized-types.md)
- [ffi-wrap-each-call](ffi-wrap-each-call.md)
