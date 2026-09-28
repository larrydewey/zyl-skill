# boot-parens-per-file

> Keep every top-level form independently balanced and starting in column 1; check every edited file with `zyl balance`.

## Why It Matters

A missing closer silently nests every following `defn` inside the broken form; they vanish from compiled output. The balance check (`sexp_balance.zyl`, run first by `compile-check-balance` and by `zyl-parse`) catches net imbalance in a source file, with line/col and a fix-it. Every compiler module is its own file and is checked on its own by any compile that reaches it, including `./boot.sh`. Since 2026-09-28 it also catches a misplaced paren that leaves the file net-balanced, through the column-1 layout rule (spec §1.6, [tool-balance](tool-balance.md)). (Until 2026-09-24 the compiler was built from one concatenated bundle whose depth check was whole-bundle only; a 14-paren deficit in `error_codes.zyl` shipped that way.)

## Good

```bash
build/boot/zyl-self balance stdlib/compiler                # every module, in about 0.1 s
./run_regression_tests.sh --full --no-boot --filter balance # the checker's own tests
```

## Symptoms

- Source files are read whole; the old 1 MiB cap that truncated a larger module is gone.
- The common net-balanced shape, a `defn` parameter list swallowing its body, is `E_MALFORMED_PARAMETER`.

## See Also

- [syn-brackets-and-balance](syn-brackets-and-balance.md)
- [boot-module-build](boot-module-build.md)
