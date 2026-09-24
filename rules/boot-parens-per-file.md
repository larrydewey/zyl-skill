# boot-parens-per-file

> Keep every top-level form independently balanced; a missing closer swallows everything after it in the same file.

## Why It Matters

A missing closer silently nests every following `defn` inside the broken form; they vanish from compiled output. The balance check (`sexp_balance.zyl`, run first by `compile-check-balance` and by `zyl-parse`) catches net imbalance in a source file, with line/col and a fix-it. Every compiler module is its own file and is checked on its own by any compile that reaches it, including `./boot.sh`. It does **not** catch a misplaced paren that leaves the file net-balanced. (Until 2026-09-24 the compiler was built from one concatenated bundle whose depth check was whole-bundle only; a 14-paren deficit in `error_codes.zyl` shipped that way.)

## Good

```bash
./boot.sh                                                  # balance error reported first, with file:line:col
./run_regression_tests.sh --full --no-boot --filter balanced-parens
```

## Symptoms

- A function "missing from compiled output", or `E_UNBOUND_VARIABLE` for a function you can see: check balance of the forms **before** it.
- The common net-balanced shape, a `defn` parameter list swallowing its body, is `E_MALFORMED_PARAMETER`.

## See Also

- [syn-brackets-and-balance](syn-brackets-and-balance.md)
- [boot-module-build](boot-module-build.md)
