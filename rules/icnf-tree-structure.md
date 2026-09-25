# icnf-tree-structure

> Treat ICNF as an untyped structured tree of 21 `Icnf` constructors (not SSA), with three parameter representation kinds and every analysis result in side tables keyed by node.

## Why It Matters

The spec calls ICNF an SSA IR with per-value regions; the implementation is a tree, not SSA, and facts about nodes live in side tables keyed by node address (`node_tables.zyl`), not in the nodes. Passes must be written for what exists. The native backend's MIR ([cg-native-backend-mir](cg-native-backend-mir.md)) is the only linear, register-level IR, and it is built per function inside codegen.

```lisp
(deftype Icnf
  (IConst Int) (IStr String) (IFlt String) (ILoad String)
  (IBinop Int Icnf Icnf) (ICall String (List Icnf)) (IFfi String (List Icnf))
  (IPrint Icnf) (IIf Icnf Icnf Icnf) (IWhile Icnf Icnf) (ISet String Icnf)
  (ILet String Icnf Icnf) (ISeq (List Icnf))
  (IVariant String Int (List Icnf)) (IMatch Icnf (List IArm))
  (IFn String (List String) Icnf (List Int))   ; name params body param-kinds
  (ICallClosure String (List Icnf))            ; no longer produced
  (ITryCatch Icnf String Icnf)
  (IStackVariant String Int (List Icnf))
  (ISymAddr String)                            ; C symbol address via GOT; only from ic-ffi
  (IRegion Int Int Int Int Icnf))              ; with-region: kind (1 arena, 2 fixed) block align limit body
(deftype IArm (IArm String Int (List String) Icnf))  ; variant tag binds body; wildcard tag -1
```

A program is a `(List Icnf)` of `IFn`s (including lifted lambdas and `_test_<name>` functions).

## Binop opcodes

0 `+`, 1 `-`, 2 `*`, 3 `/`, 4 `%`, 5 `<`, 6 `>`, 7 `<=`, 8 `>=`, 9 `=`/`==`, 10 `!=`, 11 `bit-and`, 12 `bit-or`, 13 `bit-xor`, 14 `shl`, 15 `shr`, 16 `ashr`. `(bit-not x)` is `(IBinop 13 x (IConst -1))`.

## Notes

- Types are erased; each `IFn` carries param kinds (0 word, 1 String, 2 Float).
- Side tables on `Icnf` nodes (`zyl_attrh_*`): `icnf-kinds` (codegen kind: 1 String, 2 Float, 3 variant), `icnf-regions` (region inference: site 1 frame, 2 result, 3 heap, `4 + k` with-region scope `k`; on an `IFn`, flags + 4), `icnf-scalars` (typed Int, Bool or Float), `icnf-adts` (typed as a program ADT, a heap block with a size header), `icnf-reuse` (on an `IVariant`: the variable whose block it may take). All are emptied per program by `node-tables-clear`.
- Textual form: `icnf_print.zyl` (canonical text, hashed by package builds). A node's close paren is preceded by ` :k` for a codegen representation kind and ` @r` for a region annotation. Reuse decisions are not printed; the owning clones they add (`f~own`) are ordinary `IFn`s and are.
- Consumers, in order: inliner and copy propagation, folding, region inference, the reuse pass, codegen (native backend or stack machine per function); also the REPL interpreter (`stdlib/repl/interp.zyl`), which ignores regions and reuse.
- `ICallClosure` is kept by `ic-hoist` and walked by every pass but no longer produced by lowering: a call through a local is an `ICall` on the local's name.

## See Also

- [icnf-new-form-needs-case](icnf-new-form-needs-case.md)
- [icnf-lowering-map](icnf-lowering-map.md)
