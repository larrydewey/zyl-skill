# boot-lifted-constraints

> Know which historical bootstrap constraints are lifted, so you neither follow dead rules blindly nor reintroduce the patterns they guarded against.

## Lifted (still verified by the fixed point)

| Old rule | Status |
|---|---|
| Function arity ≤ 6 | lifted 2026-08-25: stack-passed args work (`cg-call-args` scratch staging, `cg-param-spills`). Prefer ≤ 6 for readability. |
| No `match` in value position | lifted 2026-08-25: let values, binop args, if branches, call args, nested arm bodies (`tests/regression/match-value-position.zyl`). Prefer flat code. |
| No cross-module shared list helpers | lifted 2026-08-25: per-site generic inference works (`list-head-or` shared in codegen.zyl). |
| `;` inside strings truncates | fixed: the lexer is string-aware (older stage binaries truncated). |
| `d1`/`d2` dummy names instead of `_` | gone: `_` is the discard everywhere; wildcard arms are fine. |
| Two calls in one binop compute 0 | no longer reproduces; narrowed to `E_MATCH_ARM_COMPLEX` ([boot-bind-calls-before-binop](boot-bind-calls-before-binop.md)). |

## Still active (old SKILL.md §2 numbering)

| # | Rule |
|---|---|
| 3 | [match-misspelled-last-arm](match-misspelled-last-arm.md) |
| 4 | [boot-parens-per-file](boot-parens-per-file.md) |
| 5 | [boot-one-deftype-per-name](boot-one-deftype-per-name.md) |
| 7 | [proj-buf-append-appends](proj-buf-append-appends.md) |
| 8 | [boot-bind-calls-before-binop](boot-bind-calls-before-binop.md) |
| 10 | [boot-moderate-bodies](boot-moderate-bodies.md) |
| 11 | [boot-even-field-parity](boot-even-field-parity.md) |
| 12 | [pkg-library-no-main](pkg-library-no-main.md) |
| 13 | [pkg-use-what-you-construct](pkg-use-what-you-construct.md) |

Violations usually show up as a miscompile in stage 2 or later, not as an error. Update this file whenever a constraint changes: a stale rule reads as authoritative.

## See Also

- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
