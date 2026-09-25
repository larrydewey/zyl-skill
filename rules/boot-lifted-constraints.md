# boot-lifted-constraints

> Know which historical bootstrap constraints are lifted, so you neither follow dead rules blindly nor reintroduce the patterns they guarded against.

## Lifted (still verified by the fixed point)

| Old rule | Status |
|---|---|
| Function arity ≤ 6 | lifted 2026-08-25: stack-passed args work (`cg-call-args` scratch staging, `cg-param-spills`). Functions with more than 6 parameters, or calls with more than 6 arguments, stay on the stack machine; prefer ≤ 6 for readability and speed. |
| No `match` in value position | lifted 2026-08-25: let values, binop args, if branches, call args, nested arm bodies (`tests/regression/match-value-position.zyl`). Prefer flat code. |
| No cross-module shared list helpers | lifted 2026-08-25: per-site generic inference works; the type pass specializes per type (`f~T`). |
| `;` inside strings truncates | fixed: the lexer is string-aware (older stage binaries truncated). |
| `d1`/`d2` dummy names instead of `_` | gone: `_` is the discard everywhere; wildcard arms are fine. |
| Two calls in one binop compute 0 | re-tested 2026-09-25 in both backends: correct everywhere; only the match-arm shape with a literal is still rejected (`E_MATCH_ARM_COMPLEX`, [boot-match-arm-call-sums](boot-match-arm-call-sums.md)). |
| Even field count for records built with nested constructions | re-tested 2026-09-25: odd counts are correct ([boot-field-parity-lifted](boot-field-parity-lifted.md)); `sexp_balance.zyl`'s even `CheckState` is a leftover. |
| Bitwise operators cannot be used in compiler source | gone: `mir.zyl` uses `bit-and`, `bit-or`, `shl`, `shr`; the optimizer still does not fold them, by choice. |
| Compare built strings with `str-eq`, not `=` | `=` compares String contents wherever the type is String, which sound typing now always knows ([pass-string-eq-in-compiler](pass-string-eq-in-compiler.md)). |
| Two definitions of one type name miscompile silently | now a type error (`cannot unify Shape with Shape`), still to be avoided ([boot-one-deftype-per-name](boot-one-deftype-per-name.md)). |

## Still active (old SKILL.md §2 numbering)

| # | Rule |
|---|---|
| 3 | [match-misspelled-last-arm](match-misspelled-last-arm.md) |
| 4 | [boot-parens-per-file](boot-parens-per-file.md) |
| 5 | [boot-one-deftype-per-name](boot-one-deftype-per-name.md) |
| 7 | [proj-buf-append-appends](proj-buf-append-appends.md) |
| 8 | [boot-match-arm-call-sums](boot-match-arm-call-sums.md) (the match-arm shape only) |
| 10 | [boot-moderate-bodies](boot-moderate-bodies.md) |
| 12 | [pkg-library-no-main](pkg-library-no-main.md) |
| 13 | [pkg-use-what-you-construct](pkg-use-what-you-construct.md) |
| new | [boot-two-step-syntax](boot-two-step-syntax.md) for new syntax and for runtime functions the compiler calls |

The type checker now reports most of what used to surface as a stage-2 miscompile, but a violation of a rule above can still show up as wrong output rather than an error. Update this file whenever a constraint changes: a stale rule reads as authoritative.

## See Also

- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
