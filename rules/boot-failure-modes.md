# boot-failure-modes

> Map a bootstrap failure to its cause before changing anything.

| Symptom | Likely cause | Action |
|---|---|---|
| `reproduced asm differs from committed seed` | compiler source changed its own output | reseed ([boot-fixed-point-workflow](boot-fixed-point-workflow.md)) |
| `FIXED POINT BROKEN` | non-determinism, or a behavior change needing reseed | diff stage2.s/stage3.s; see below |
| stage 2 crashes | stage 1 miscompiled the compiler | bisect with small inputs |
| a later file's definitions vanish | paren imbalance in an earlier file | compile the file alone |
| undefined `_ZYL_<Ctor>` only when compiled standalone | missing `use` hidden by the bundle | add the `use` |
| boot very slow | repeated subtree work in a pass | [pass-avoid-repeated-subtree-work](pass-avoid-repeated-subtree-work.md) |

## Reading a fixed-point diff

```bash
diff -u build/boot/stage2.s build/boot/stage3.s | head -100
```

- Function labels differ (`zy_...`, `_lambda_N`) → naming: monomorphization, lambda lifting, fresh ids.
- Instructions differ inside a function → ICNF lowering, optimization, codegen.
- `.rodata` differs → string/float constants.
- Whole functions appear/disappear → module resolution, macro expansion, or paren imbalance.

## Bisecting

```bash
build/boot/stage1.bin small.zyl -o /tmp/s1.s --emit-asm
build/boot/stage2.bin small.zyl -o /tmp/s2.s --emit-asm
diff /tmp/s1.s /tmp/s2.s
```

Two binaries from the same source differing = behavior change needing reseed. The **same** binary differing across two runs = real non-determinism (address-derived names, pointer string compare, hash iteration). `ZYL_DEBUG_STAGES=1` shows which phase a compile reached. Shrink to one construct and add it to `tests/regression/`.

## See Also

- [det-no-address-dependent-output](det-no-address-dependent-output.md)
