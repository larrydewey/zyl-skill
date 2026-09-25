# boot-failure-modes

> Map a bootstrap failure to its cause before changing anything.

| Symptom | Likely cause | Action |
|---|---|---|
| `reproduced asm differs from committed seed` | compiler source changed its own output | reseed ([boot-fixed-point-workflow](boot-fixed-point-workflow.md)) |
| `FIXED POINT BROKEN` | non-determinism, or a behavior change needing reseed | diff stage2.s/stage3.s; see below |
| `error[E_TYPE_MISMATCH]` ... `the program does not type-check` in stage 2 | the compiler source itself is ill-typed (it is checked like any program) | fix the source; every error is listed before the stage fails |
| `E_CANNOT_INFER` (`untyped ffi result`, or `ffi-call to ... which has no (extern ...) declaration`) in stage 2 | compiler source calls a runtime function the seed has no signature for, or one missing from the runtime's `X(...)` table | two steps ([boot-two-step-syntax](boot-two-step-syntax.md)) |
| stage 2 crashes | stage 1 miscompiled the compiler | bisect with small inputs; then with the switches below |
| a file's later definitions vanish | a missing closer earlier in that file | read the balance error's opener |
| `E_UNBOUND_VARIABLE` for a function another module defines | missing `use` | add the `use` |
| `E_OUT_OF_MEMORY` in a stage | a change allocates far more (e.g. strings built inside a lookup) | [pass-no-allocation-in-lookups](pass-no-allocation-in-lookups.md); don't just raise `ZYL_STAGE_MEMORY` |
| boot very slow | repeated subtree work in a pass, or a fixpoint that no longer converges quickly | [pass-avoid-repeated-subtree-work](pass-avoid-repeated-subtree-work.md) |

## Reading a fixed-point diff

```bash
diff -u build/boot/stage2.s build/boot/stage3.s | head -100
```

- Function labels differ (`zy_...`, `_lambda_N`, `f~T` instances, `f~own` clones) → naming: type-pass specialization, lambda lifting, fresh ids, the reuse pass.
- Instructions differ inside a function → ICNF lowering, optimization, region inference, codegen. Register names differing inside a native-backend function (`.L<k>_<n>` labels) → the MIR lowering or allocator; its choices must depend on instruction order alone.
- `.rodata` differs → string/float constants.
- Whole functions appear/disappear → module resolution, macro expansion, inlining, or paren imbalance.

## Bisecting

```bash
build/boot/stage1.bin small.zyl -o /tmp/s1.s --emit-asm
build/boot/stage2.bin small.zyl -o /tmp/s2.s --emit-asm
diff /tmp/s1.s /tmp/s2.s
```

Two binaries from the same source differing = behavior change needing reseed. The **same** binary differing across two runs = real non-determinism (address-derived names, pointer string compare, hash iteration). `ZYL_DEBUG_STAGES=1` appends each phase name to `/tmp/dbg` as the compile reaches it. Shrink to one construct and add it to `tests/regression/`.

Compile-time switches that turn one transformation off, for bisecting a miscompile (each changes the output, so use them on test programs, not on a stage you want to compare with the seed):

| Switch | Turns off |
|---|---|
| `ZYL_MIR=0` | the native backend: every function through the stack machine |
| `ZYL_INLINE=0` | inlining and copy propagation |
| `ZYL_REUSE=0` | in-place reuse and owning clones |
| `ZYL_REGIONS=0` | region placement: every site on the heap |

## See Also

- [det-no-address-dependent-output](det-no-address-dependent-output.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
