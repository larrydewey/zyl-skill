# cg-native-backend-mir

> Most functions are compiled by the native backend: ICNF lowered to MIR (`ml-*`), liveness and linear-scan register allocation (`mir.zyl`), emission (`mb-*`). Extend it by widening `ml-ok` and `ml-expr` together; anything `mb-eligible` rejects falls back to the stack machine.

## Why It Matters

Since 2026-09-25 `cg-function` asks `mb-eligible` first, and most functions (about 95% of the compiler's own when last counted, 2026-09-25) go through the native backend; the rest, and every function under `ZYL_MIR=0` at compile time, go through the stack machine ([cg-stack-machine-fallback](cg-stack-machine-fallback.md)). Both share one ABI (SysV argument registers, result in `rax`, the region frame words, callee-saved registers preserved; a leaf native function with no spills, blocks, regions or C calls is frameless, with no `push rbp`; one that calls keeps `rbp` (push, mov, pop) so the panic backtrace can walk the `rbp` chain, [cg-symbols-and-entry](cg-symbols-and-entry.md)), so they call each other freely. Design: `docs/native-backend-design.md`.

## Pipeline, per function

| Step | Where | What |
|---|---|---|
| eligibility | `mb-eligible`, `ml-ok` (codegen.zyl) | not a `Secret`-wiping function, at most 6 parameters, and every node in the supported set |
| lowering | `ml-function`, `ml-expr`, `ml-tail` | a linear list of `MI` instructions over virtual registers (vregs numbered in lowering order), labels local to the function |
| liveness | `mir-liveness` | backward fixpoint over bit sets, one bit per vreg |
| intervals | `mir-intervals` | start/end per vreg, and whether a call lies inside (`mi-is-call`) |
| allocation | `mir-allocate` (linear scan, Poletto–Sarkar) | sorted by start then vreg; a value live across a call gets a callee-saved register; move and parameter hints; the interval ending last is spilled |
| emission | `mb-emit-fn`, `mb-emit-one` | prologue, the parameters as one parallel move (`pm-sequence`), one instruction at a time |

Every step is a function of instruction order alone, so the same function always gets the same registers: the fixed point depends on it.

## Supported set (`ml-ok`)

`IConst`, `IStr`, `IFlt`, `ISymAddr`; `ILoad` of a local or a known function (`MFnRef`); `IBinop` 0–16 and the wrapping and saturating opcodes 18–23 on word operands only (`cg-int-operands`); checked `+ - *` get a `jo` to the overflow trap (`mb-checked-tail`); `ICall` of a known top-level function (not a local) with at most 6 arguments; `IFfi` with at most 6 arguments (byte access and `Array` get/set/cap are inlined); `IIf`, `IWhile`, `ISet` of a local, `ILet`, `ISeq`; `IVariant`, `IStackVariant`, `IMatch`. Allocation and call sites must have region level ≤ 3 (no `with-region` scope). Everything else (`IPrint`, `ITryCatch`, `IRegion`, `IFn`, `ICallClosure`, calls through locals, Float/String/ADT operators) keeps the whole function on the stack machine.

## What the emitted code does

- Registers: [cg-callee-saved-registers](cg-callee-saved-registers.md). Frames: saved callee-saved registers, spill slots, six staging words when a tail call must survive a region release, stack-variant blocks; 48 region bytes on top when the function has region flags (the stack machine's layout). Alignment: [cg-c-call-alignment](cg-c-call-alignment.md).
- A self tail call with matching arity is a jump to the loop head after label 0, the arguments assigned by moves (an argument that is its own parameter unchanged is skipped); a frame region is recycled (`MRegionCycle`, `zyl_region_recycle`) rather than released. Other tail calls of user functions become `MTail`: moves, restore, `jmp`.
- Inline operations: byte loads and stores with the runtime's handle and bound checks (a byte-buffer parameter that is never `set!` and passed unchanged by every self call has its data pointer and bound loaded once, `MByteData`/`MByteBound`); `Array` access with the slow path calling the runtime entry; region bump allocation with a runtime slow path; division and remainder by a literal other than 0 and -1 through `zyl_div_magic`/`zyl_div_shift` (`mb-divmod-const`), and by anything else with the zero-divisor and `-1` tests before `idiv` ([cg-call-arg-staging](cg-call-arg-staging.md)).
- A variant construction the reuse pass marked (`icnf-reuse`) becomes `MReuse`: the old block when its size header is large enough, else a fresh allocation ([icnf-reuse-pass](icnf-reuse-pass.md)).
- Evaluation order: a local read is copied into a fresh vreg when a later sibling could `set!` it ([pass-evaluation-order-sets](pass-evaluation-order-sets.md)).

## Bad

An arm added to `ml-ok` with no matching case in `ml-expr`: the new node passes eligibility, and `ml-expr`'s catch-all `(_ (ml-const ms 0))` lowers it to the constant 0, silently.

```text
(IFoo x (ml-ok st env x))        ; in ml-ok only
```

## Good

Change `ml-ok` and `ml-expr` (and `ml-tail`, if the node can be in tail position) together. A new MIR instruction also gets `mi-uses` and `mi-def` cases, `mi-is-call` if it calls anything, `mb-any-c-call` if it calls C, `mi-pure` if it has no effect and cannot trap (checked arithmetic and division trap, so they are not pure), and an `mb-emit-one` case.

## Notes

- `ZYL_MIR=0` at compile time sends every function to the stack machine: bisect a suspected backend bug with it before anything else.
- Emission uses only `rax rcx rdx r11` as scratch; everything else may hold a live vreg.
- Labels are `<base>_<n>` where `<base>` is a fresh `cg-label-new` label per function (`.L221_0` is the loop head); inline sequences add suffixes (`_m`, `_u`, `_a`, `_b`, `_q`, `_v`, `_c`, `_rr`).

## See Also

- [cg-stack-machine-fallback](cg-stack-machine-fallback.md)
- [cg-callee-saved-registers](cg-callee-saved-registers.md)
- [cg-c-call-alignment](cg-c-call-alignment.md)
- [cg-call-arg-staging](cg-call-arg-staging.md)
- [icnf-optimizer-scope](icnf-optimizer-scope.md)
