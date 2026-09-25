# icnf-optimizer-scope

> Expect four safe ICNF optimizations (small-function inlining, copy propagation of let-bound locals, integer constant folding for opcodes 0–10, dead-branch elimination), then in-place reuse after region inference; never add an optimization that reorders or drops effects.

## Why It Matters

Safe-only optimization is an architecture decision. What exists, in pipeline order (`lower-after-mono` in `pipeline.zyl`):

| Pass | Where | Does |
|---|---|---|
| inlining | `opt-inline-fns` (optimization.zyl), two rounds | replaces a call of a small function by its body: arguments bound first, in order, by nested `let`s, every binder in the copy renamed (`__in<id>_<name>`) |
| copy propagation | `opt-copyprop-fns`, after inlining | `(let n x body)` with `x` a local read becomes `body` with `n` replaced by `x`, when neither is `set!` in `body` and `body` binds no `x` |
| folding + dead branches | `opt-optimize-fns`, one bottom-up walk | integer arithmetic and comparisons on literals: `(+ (* 2 3) 4)` becomes `(IConst 10)`; `(IIf (IConst 1) a b)` becomes `a`, `(IIf (IConst 0) a b)` becomes `b`; `(IWhile (IConst 0) body)` becomes `(IConst 0)` |
| in-place reuse | `ru-reuse` (reuse.zyl), after `rg-regions` | marks constructions that may take a dead, unique value's block ([icnf-reuse-pass](icnf-reuse-pass.md)) |

Inlining candidates: not `main`, parameters all of word kind (a String or Float parameter's kind would become its argument's), no self call, only node kinds a copy reproduces exactly (no `try`, region scope, lambda, closure call or `print`), and at most `ZYL_INLINE_LIMIT` nodes (default 6). A leaf (no user calls) may be three times that, but is inlined only into a function that calls itself (a loop): inlining such leaves everywhere grew the compiler by 40%. A call is left alone when its name is a local at the site, or when the body names a function or global a local at the site would shadow. Inlining runs before region inference, which places the inlined allocations like any others. `ZYL_INLINE=0` turns off inlining and copy propagation.

Not done: folding division or remainder by a constant zero (it still fails at run time), floats (`IFlt` is source text), bitwise opcodes 11–16 (no seed limitation remains, `mir.zyl` itself uses `bit-and`; nobody has added them), a constant-true `while`, DCE of unused lets, CSE, loop-invariant motion.

## Where other optimizations live

- Division and remainder by a constant without `idiv`, immediate operands, compare-and-branch, and self tail calls as jumps are the native backend's ([cg-native-backend-mir](cg-native-backend-mir.md)); the stack machine turns direct tail calls into jumps too.
- Stack promotion of let-bound variants (`IStackVariant`) and region placement are region inference ([icnf-regions-are-a-rewrite](icnf-regions-are-a-rewrite.md)).

## Notes

- Every rebuilt node goes through `opt-keep` (kind, span, scalar and ADT marks): [pass-keep-kinds](pass-keep-kinds.md).
- Copy propagation removes only a load, which has no effect; inlining keeps the call's argument order. Neither moves a side effect.

## See Also

- [det-left-to-right](det-left-to-right.md)
- [icnf-reuse-pass](icnf-reuse-pass.md)
- [pass-adding-a-pass](pass-adding-a-pass.md)
