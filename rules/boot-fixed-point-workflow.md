# boot-fixed-point-workflow

> After editing `stdlib/compiler/*.zyl`, `selfhost/`, `runtime/rt/` or a stdlib module the driver `use`s (the REPL and LSP modules included): reseed, verify, commit the seeds.

## Why It Matters

The compiler is compiled by the previous generation of itself and must reproduce its committed output byte for byte. A source change that alters the compiler's own output makes `./boot.sh` fail with `reproduced asm differs from committed seed` (stage 2), a runtime change `reproduced runtime differs from committed rt.s`, and non-determinism `FIXED POINT BROKEN` (stage 3), and nothing else will trust the new seed until it is reseeded. With inlining, the native backend's register allocation and the reuse pass, almost any compiler change alters its own output.

## The workflow

```bash
./boot.sh --bootstrap-from-self    # iterate stageN -> stageN+1 until two rounds agree (<= 10)
./boot.sh                          # verify a clean fixed point on the new seed
./run_regression_tests.sh --full --no-boot
git add -f build/boot/stage2.s build/boot/stage2.bin build/boot/rt.s && git commit
```

A verified `./boot.sh` ends by refreshing an existing install (`~/.zyl`, or `$ZYL_INSTALL_HOME`) with `uninstall.sh` + `install.sh`; `ZYL_NO_INSTALL_REFRESH=1` skips it.

## What `./boot.sh` checks

```
0. copies stdlib/ and runtime/rt/ into build/boot/ (what every stage resolves)
   and assembles the committed runtime seed build/boot/rt.s -> rt.o
1. cc links committed build/boot/stage2.s + rt.o            -> stage1.bin
2. stage1.bin compiles selfhost/driver.zyl --emit-asm      -> stage2_gen.s   (cmp vs stage2.s)
   stage1.bin compiles runtime/rt/rt.zyl --runtime-module   (cmp vs rt.s; every zyl_* defn emitted)
3. cc links stage2.s                                        -> stage2.bin
4. stage2.bin compiles the driver and the runtime again    -> stage3.s       (cmp vs stage2.s, rt.s)
5. stage2.bin rt-cache                                      -> rt.zo (the Zyl linker's runtime)
6. smoke program must print 42 and 3
7. writes the zyl-self wrapper and builds zyl-lsp in build/boot/, then refreshes the install
```

Comparisons are `cmp` on **assembly**. The compiler stages link hosted with `cc` (the REPL interpreter's FFI uses `dlsym`); the programs they compile link freestanding through the Zyl assembler and linker. In `--bootstrap-from-self`, each round's runtime is compiled by that round's compiler, so generated code and the runtime always agree on shared formats (the try frame's layout and pointer mangling). Each stage runs under `ZYL_STAGE_TIMEOUT` (default 2400 s) and an allocation ceiling `ZYL_STAGE_MEMORY` (default 4 GB, passed on as `ZYL_MAX_MEMORY`; cumulative bytes allocated, nothing is freed during a compile). A self-compile allocates somewhat over 2 GB and takes about 2.2 s, so `E_OUT_OF_MEMORY` in a stage means a compiler change allocates far more than before ([pass-no-allocation-in-lookups](pass-no-allocation-in-lookups.md)). A full verification takes well under a minute. `boot.sh` exports `ZYL_HOME=build/boot`.

## Two steps for a new runtime function

The seed type-checks the compiler source with **its own** copy of `ffi_sigs.zyl`. So a runtime function the compiler itself (or the REPL, LSP or any module the driver reaches) will call lands in two reseeds:

1. Add the `zyl_*` defn to `runtime/rt/` and its signature to `ffi_sigs.zyl` (and to `ffi-raw-p` if only the standard library may call it). Do not call it yet. Reseed.
2. Now call it; reseed again.

Calling it in step 1 fails stage 2 with `E_CANNOT_INFER` (`no type for ffi-call to ..., which has no (extern ...) declaration`). The interpreter reaches a runtime entry only if `runtime/rt/ffitab.zyl` lists it.

## Notes

- `--bootstrap-from-self` converges because a compiler that just compiled a behavior change does not yet exhibit it; the next round does.
- The fixed point proves determinism and self-consistency, not correctness for programs unlike the compiler: the regression suite (with interpreter differential) covers the rest.

## See Also

- [boot-two-step-syntax](boot-two-step-syntax.md)
- [boot-failure-modes](boot-failure-modes.md)
- [boot-module-build](boot-module-build.md)
