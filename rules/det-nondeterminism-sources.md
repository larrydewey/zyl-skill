# det-nondeterminism-sources

> Keep clock, environment, PID, kernel entropy and foreign output out of anything that must be reproducible — the compiler will not warn you.

## Why It Matters

A program that stays out of the FFI is deterministic in its output, actors included: channels form a Kahn network and each actor's output is buffered until it is joined ([actor-output-per-actor](actor-output-per-actor.md)). These are the known escapes; under §27 anything else is a bug.

| Source | How it enters |
|---|---|
| Foreign C writing to stdout directly | a C `write` bypasses the runtime's per-actor output buffers |
| Heap addresses | printing a value that has no `Show`, comparing handles ([det-no-address-dependent-output](det-no-address-dependent-output.md)) |
| Clock, PID, environment | `ffi-call` to the runtime's `zyl_now_ms` or `zyl_getenv_str`, or to C's `time`/`getpid`/`getenv` (each declared with `(extern ...)`) |
| FFI timeouts | whether a foreign call finishes inside its timeout or raises `E_FFI_TIMEOUT` depends on wall-clock time |
| Kernel entropy | `math/rand/crypto` (`sysrng-*`) |
| Stdlib location | stale `~/.zyl` shadows the checkout when `ZYL_HOME` is unset |
| Floating point | deterministic in practice (no FMA, no `-march`), not a checked rule |

## Good

```lisp
(use math/rand/deterministic)
(defn main ()
  (let rng (chacharng-from-int 42)            ; the same seed gives the same stream
    (begin
      (print (if (= (chacharng-next-u64 rng) (chacharng-next-u64 (chacharng-from-int 42))) "same" "differs"))   ; same
      0)))
```

## Notes

- Same source → identical assembly (independent of directory and `-o`), and an identical binary: freestanding programs are assembled and linked by the compiler itself, byte-deterministically; a hosted link is identical for the same `cc` (the `.file` directive uses the basename so linker temp names don't leak in).
- Package builds: same `zyl.pkg` + `zyl.lock` + compiler ⇒ same binary; network only in `zyl fetch`.

## See Also

- [actor-output-per-actor](actor-output-per-actor.md)
- [pkg-stdlib-resolution](pkg-stdlib-resolution.md)
