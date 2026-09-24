# pass-diagnostics

> Report new errors through `err-at` with a node and a catalogued code from `error_codes.zyl`, so they print located with a help line.

## Why It Matters

`error_codes.zyl` is the single-source catalog (name, phase, severity 1 error / 2 warning, default message). `error_report.zyl` has the formatting helpers: `err-header`, `int-to-str`, `loc-string`, and `err-at`, which renders `error[CODE]: msg`, `--> file:line:col`, the source line, a caret and `= help:` from a node's recorded offset. Located today: `E_MALFORMED_PARAMETER`, balance errors, `E_ARITY_MISMATCH`, `E_NON_EXHAUSTIVE_MATCH`, `E_UNREACHABLE_MATCH_ARM`, `E_DUPLICATE_DEFINITION`, `E_UNBOUND_VARIABLE`, the closure-capture `E_MUT_CONFLICT`. Mutability, capability, unused, secret and most `expr_inner` errors still print a bare `PANIC:` line.

## Good

- Add the code to `error_codes.zyl` (and spec §28 if it is a language error).
- Thread the offending node to `err-at`; include a concrete `= help:` fix.
- Colocate fix-it text with the check (as `sb-hint` is with `sb-check-string`) so CLI, LSP and REPL share wording.
- Add a `tests/compile-fail/` case and, ideally, a regression test asserting location and hint.

## Notes

- Still open: colorized output, "did you mean?" suggestions (`docs/error-system-architecture.md`).
- Every check stops at its first error.

## See Also

- [pass-copy-spans](pass-copy-spans.md)
- [reference: error codes](../references/error-codes.md)
