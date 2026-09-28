# boot-runtime-module

> The runtime is Zyl (`runtime/rt/*.zyl`, entry `rt.zyl`), compiled with `--runtime-module`: only there do the locked `%` primitives exist and `zyl_*` defns become exported labels. Change it like compiler source: reseed, and land any entry the compiler calls in two steps.

## Why It Matters

There is no C runtime and no libc under a freestanding program (`docs/runtime-in-zyl-design.md`). The runtime's own compile is the only place raw memory, syscalls and SIMD are reachable, which is how programs get none of it:

- The driver refuses `--runtime-module` unless the entry is the bundle's `runtime/rt/rt.zyl`. A runtime module may `use` only its siblings; a stdlib module is refused at every level (`tests/scripts/runtime-module-lock.sh`).
- `%load8`..`%store64`, `%syscall0`..`%syscall6`, `%cas`, `%tls`, `%global`, `%fn`, `%call0`..`%call16`, `%v128-*`/`%v256-*` and the rest are `E_FFI_RESTRICTED` anywhere else, the stdlib and the compiler included.
- In runtime mode there is no `main`, no prelude, no region inference; a top-level `defn` named `zyl_*` keeps that bare label and is `.globl`.
- Programs reach an entry through codegen or a typed `ffi-call "zyl_..."` (`ffi_sigs.zyl`); raw-word entries are listed in `ffi-raw-p` and callable only from the stdlib.

## Pitfalls

- An uncalled Num-generic `zyl_*` function is never instantiated, so it is not emitted; `boot.sh` fails with `runtime entries not emitted`. Annotate its parameters.
- A C `int` result has only its low 32 bits defined; mask it (`rt-lo32`) before comparing.
- `shr` is logical and `ashr` arithmetic ([bits-shr-vs-ashr](bits-shr-vs-ashr.md)).
- Anything the interpreter must call needs an entry in `ffitab.zyl`.
- Shared runtime state touched from actors needs a lock once threads exist (`threads_started`), as the allocator and the interpreter's name table do.

## See Also

- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
- [boot-two-step-syntax](boot-two-step-syntax.md)
- [cg-self-link](cg-self-link.md)
