# det-nondeterminism-sources

> Keep clock, environment, PID, kernel entropy and multi-actor output out of anything that must be reproducible — the compiler will not warn you.

## Why It Matters

A program that stays out of the FFI and out of actors is deterministic in its output. These are the known escapes; under §27 anything else is a bug.

| Source | How it enters |
|---|---|
| Actor output interleaving | one pthread per actor, OS scheduling |
| Heap addresses | printing a value that has no `Show`, comparing handles ([det-no-address-dependent-output](det-no-address-dependent-output.md)) |
| Clock, PID, environment | `ffi-call` to the runtime's `zyl_now_ms` or `zyl_getenv_str`, or to C's `time`/`getpid`/`getenv` (each declared with `(extern ...)`) |
| FFI timeouts | whether a foreign call finishes inside its timeout or raises `E_FFI_TIMEOUT` depends on wall-clock time |
| Kernel entropy | `math/rand/crypto` (`sysrng-*`) |
| Stdlib location | stale `~/.zyl` shadows the checkout when `ZYL_HOME` is unset |
| Floating point | deterministic in practice (no FMA, no `-march`), not a checked rule |

## Good

```lisp
;; reproducible randomness for tests/simulations
(use math/rand/deterministic)
(let rng (chacharng-from-int arena 42) ...)
```

## Notes

- Same source → identical assembly (independent of directory and `-o`), and identical binary for the same toolchain (the `.file` directive uses the basename so linker temp names don't leak in).
- Package builds: same `zyl.pkg` + `zyl.lock` + compiler ⇒ same binary; network only in `zyl fetch`.

## See Also

- [actor-output-nondeterministic](actor-output-nondeterministic.md)
- [pkg-stdlib-resolution](pkg-stdlib-resolution.md)
