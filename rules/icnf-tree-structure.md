# icnf-tree-structure

> Treat ICNF as an untyped structured tree of `Icnf` nodes (not SSA), with three parameter representation kinds.

## Why It Matters

The spec calls ICNF an SSA IR with per-value regions; the implementation is simpler, and passes must be written for what exists.

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
  (IStackVariant String Int (List Icnf)))
(deftype IArm (IArm String Int (List String) Icnf))  ; variant tag binds body; wildcard tag -1
```

A program is a `(List Icnf)` of `IFn`s (including lifted lambdas and `_test_<name>` functions).

## Binop opcodes

0 `+`, 1 `-`, 2 `*`, 3 `/`, 4 `%`, 5 `<`, 6 `>`, 7 `<=`, 8 `>=`, 9 `=`/`==`, 10 `!=`, 11 `bit-and`, 12 `bit-or`, 13 `bit-xor`, 14 `shl`, 15 `shr`, 16 `ashr`. `(bit-not x)` is `(IBinop 13 x (IConst -1))`.

## Notes

- Types are erased; each `IFn` carries param kinds (0 word, 1 String, 2 Float).
- No textual form/printer: inspect by writing a small program that `use`s `compiler/icnf`.
- Consumers: optimizer → region inference → codegen, and the REPL interpreter (`stdlib/repl/interp.zyl`).

## See Also

- [icnf-new-form-needs-case](icnf-new-form-needs-case.md)
- [icnf-lowering-map](icnf-lowering-map.md)
