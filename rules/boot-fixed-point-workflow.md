# boot-fixed-point-workflow

> After editing `stdlib/compiler/*.zyl`, `selfhost/` or `runtime/actor_runtime.c`: reseed, verify, commit the seed.

## Why It Matters

The compiler is compiled by the previous generation of itself and must reproduce its committed output byte for byte. A source change that alters the compiler's own output makes `./boot.sh` fail with `reproduced asm differs from committed seed` (stage 2) or `FIXED POINT BROKEN` (stage 3), and nothing else will trust the new seed until it is reseeded. With inlining, the native backend's register allocation and the reuse pass, almost any compiler change alters its own output.

## The workflow

```bash
./boot.sh --bootstrap-from-self    # iterate stageN -> stageN+1 until two rounds agree (<= 10)
./boot.sh                          # verify a clean fixed point on the new seed
./run_regression_tests.sh --full --no-boot
git add -f build/boot/stage2.s build/boot/stage2.bin && git commit
```

A verified `./boot.sh` ends by refreshing an existing install (`~/.zyl`, or `$ZYL_INSTALL_HOME`) with `uninstall.sh` + `install.sh`; `ZYL_NO_INSTALL_REFRESH=1` skips it.

## What `./boot.sh` checks

```
0. copies stdlib/ and actor_runtime.{c,h} into build/boot/ (what every stage resolves)
   and compiles build/boot/actor_runtime.o once, cc -O2
1. cc links committed build/boot/stage2.s + actor_runtime.o -> stage1.bin
2. stage1.bin compiles selfhost/driver.zyl --emit-asm      -> stage2_gen.s   (cmp vs stage2.s)
3. cc links stage2.s                                        -> stage2.bin
4. stage2.bin compiles selfhost/driver.zyl                  -> stage3.s       (cmp vs stage2.s)
5. smoke program must print 42 and 3
6. writes the zyl-self wrapper and builds zyl-lsp in build/boot/, then refreshes the install
```

Comparisons are `cmp` on **assembly**. Every stage and every program the compiler links uses the same `-O2` runtime object (`cli-link-command` in `driver.zyl` picks `actor_runtime.o` when it is newer than the source, else compiles the source with `-O2`). Each stage runs under `ZYL_STAGE_TIMEOUT` (default 2400 s) and an allocation ceiling `ZYL_STAGE_MEMORY` (default 4 GB, passed on as `ZYL_MAX_MEMORY`; cumulative bytes allocated, nothing is freed during a compile). A self-compile allocates somewhat over 2 GB, so `E_OUT_OF_MEMORY` in a stage means a compiler change allocates far more than before ([pass-no-allocation-in-lookups](pass-no-allocation-in-lookups.md)). A full verification takes well under a minute. `boot.sh` exports `ZYL_HOME=build/boot`.

## Two steps for a new runtime function

The seed type-checks the compiler source with **its own** copy of `ffi_sigs.zyl`, and links against the freshly built runtime. So a runtime function the compiler itself will call lands in two commits (`13a72eb` then `4577304` for `zyl_div_magic`):

1. Add the C function to `actor_runtime.c`, its entry to the runtime's `X(...)` table (what `zyl_runtime_export_p` and the interpreter see), and its signature to `ffi_sigs.zyl`. Do not call it from compiler source yet. Reseed and commit.
2. Now call it from the compiler; reseed again.

Calling it in step 1 fails stage 2 with `E_CANNOT_INFER` (`no type for untyped ffi result` when the entry is in the `X(...)` table but the seed has no signature; `no type for ffi-call to ...` when it is in neither).

## Notes

- `--bootstrap-from-self` converges because a compiler that just compiled a behavior change does not yet exhibit it; the next round does.
- The fixed point proves determinism and self-consistency, not correctness for programs unlike the compiler: the regression suite (with interpreter differential) covers the rest.

## See Also

- [boot-two-step-syntax](boot-two-step-syntax.md)
- [boot-failure-modes](boot-failure-modes.md)
- [boot-module-build](boot-module-build.md)
