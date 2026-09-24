# boot-fixed-point-workflow

> After editing `stdlib/compiler/*.zyl`, `selfhost/` or `runtime/actor_runtime.c`: re-bundle, reseed, verify, commit the seed.

## Why It Matters

The compiler is compiled by the previous generation of itself and must reproduce its committed output byte for byte. A source change that alters the compiler's own output makes `./boot.sh` fail with `reproduced asm differs from committed seed` (stage 2) or `FIXED POINT BROKEN` (stage 3), and nothing else will trust the new seed until it is reseeded.

## The workflow

```bash
python3 selfhost/assemble.py       # re-bundle into selfhost/zyl_selfhost_compiler.zyl
./boot.sh --bootstrap-from-self    # iterate stageN -> stageN+1 until two rounds agree (<= 10)
./boot.sh                          # verify a clean fixed point on the new seed
./run_regression_tests.sh --full --no-boot
git add -f build/boot/stage2.s build/boot/stage2.bin && git commit
./install.sh                       # if anything outside the checkout uses ~/.zyl
```

## What `./boot.sh` checks

```
1. cc links committed build/boot/stage2.s             -> stage1.bin
2. stage1.bin compiles the bundle --emit-asm          -> stage2_gen.s   (cmp vs stage2.s)
3. cc links stage2.s                                  -> stage2.bin
4. stage2.bin compiles the bundle                     -> stage3.s       (cmp vs stage2.s)
5. smoke program must print 42 and 3
6. installs stdlib/, actor_runtime.c, zyl-self wrapper, zyl-lsp into build/boot/
```

Comparisons are `cmp` on **assembly**. Each stage runs under `ZYL_STAGE_TIMEOUT` (default 2400 s); a full verification takes well under a minute. `boot.sh` exports `ZYL_HOME=build/boot`.

## Notes

- `--bootstrap-from-self` converges because a compiler that just compiled a behavior change does not yet exhibit it; the next round does.
- The stage-2 failure message still suggests `--bootstrap-from-rust`; the normal remedy is `--bootstrap-from-self`.
- The fixed point proves determinism and self-consistency, not correctness for programs unlike the compiler: the regression suite (with interpreter differential) covers the rest.

## See Also

- [boot-two-step-syntax](boot-two-step-syntax.md)
- [boot-failure-modes](boot-failure-modes.md)
- [boot-assemble-bundle](boot-assemble-bundle.md)
