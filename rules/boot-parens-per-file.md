# boot-parens-per-file

> Keep every top-level form, and every compiler-stdlib **file**, independently balanced; verify a hand-edited file by compiling it alone.

## Why It Matters

A missing closer silently nests every following `defn` inside the broken form; they vanish from compiled output. The balance check (`sexp_balance.zyl`, run first by `compile-check-balance` and by `zyl-parse`) catches net imbalance in a source file, with line/col and a fix-it. It does **not** catch a misplaced paren that leaves the file net-balanced, and `assemble.py`'s whole-bundle depth check does **not** catch a per-file deficit that another file in the bundle cancels out. A real instance shipped in `error_codes.zyl` (a 14-paren deficit in its catalog) until something finally called into it.

## Good

```bash
build/boot/zyl-self stdlib/compiler/icnf.zyl -o /tmp/icnf.s --emit-asm   # balance error reported first
./run_regression_tests.sh --full --no-boot --filter balanced-parens
```

## Symptoms

- A function "missing from compiled output", or `E_UNBOUND_VARIABLE` for a function you can see: check balance of the forms **before** it.
- The common net-balanced shape, a `defn` parameter list swallowing its body, is `E_MALFORMED_PARAMETER`.

## See Also

- [syn-brackets-and-balance](syn-brackets-and-balance.md)
- [boot-assemble-bundle](boot-assemble-bundle.md)
