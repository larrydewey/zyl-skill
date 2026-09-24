# icnf-optimizer-scope

> Expect only integer constant folding (opcodes 0–10) and dead-branch elimination; never add an optimization that reorders or drops effects.

## Why It Matters

`optimization.zyl` is two passes in one bottom-up walk. Safe-only optimization is an architecture decision.

- Folds integer arithmetic and comparisons on literals: `(+ (* 2 3) 4)` → `mov rax, 10`.
- Does **not** fold: division/remainder by constant zero (still fails at run time), floats (`IFlt` is source text), bitwise opcodes (historical seed limitation).
- `(IIf (IConst 1) a b)` → `a`; `(IIf (IConst 0) a b)` → `b`; `(IWhile (IConst 0) body)` → `(IConst 0)`; constant-true `while` is left alone.
- No DCE of unused lets, no copy propagation, no CSE, no inlining. (Direct tail calls become jumps in codegen, not in the optimizer.)

## See Also

- [det-left-to-right](det-left-to-right.md)
