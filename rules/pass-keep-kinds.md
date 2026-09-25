# pass-keep-kinds

> When a pass rebuilds an ICNF node, carry its side-table facts across (`ic-keep-kind` for the kind, `opt-keep` for kind, span, scalar and ADT marks, plus `icnf-region-set` after region inference), and never read the type pass's tables for a node it did not see.

## Why It Matters

The type pass records each `Expr` node's type in `node-types`, a call it renamed (trait method, per-type instance `f~T`, generated `T.==`) in `node-calls`, and a `print` argument's `Show` function in `node-shows`. ICNF lowering (`ic-expr`) turns the type into facts on the new `Icnf` node: `icnf-kinds` (`ic-mark-kind`, 1 String, 2 Float), `icnf-scalars` (`ta-scalar`: Int, Bool or Float) and `icnf-adts` (`ta-adt`: a program ADT, a heap block with a size header); later passes add `icnf-regions` and `icnf-reuse`. All live in `node_tables.zyl` as `zyl_attrh_*` tables keyed by node **address**: a rebuilt node starts with no entry, so its String/Float kind silently falls back to a word, region inference may put a scalar into an object class, and the reuse pass stops treating the value as an ADT block.

| Helper | Copies |
|---|---|
| `ic-keep-kind out from` (icnf.zyl) | the kind, unless `out` already has one |
| `ic-keep-span out from` | the source span |
| `opt-keep out from` (optimization.zyl) | kind, span, scalar and ADT marks |
| `ru-keep out from` (reuse.zyl) | `opt-keep` plus the region annotation |

`ic-hoist`, `opt-expr`, the inliner, copy propagation and `ri-transform-expr` all wrap their results this way.

## Good

```lisp
(defn my-pass (e)
  (opt-keep (my-pass-node e) e))       ; keeps the original's facts unless the new node has them
```

## Notes

- Same shape as span copying ([pass-copy-spans](pass-copy-spans.md)).
- The tables are emptied at the start of every `ta-annotate-program` (`node-tables-clear`), so a REPL entry never sees another entry's (freed) nodes; the parse-time field-type and extern tables are emptied at the start of `compile-to-exprs` (`field-types-clear`).
- An Expr pass that runs *after* `ta-annotate` must not rebuild nodes, or it loses their types; add such passes before it.
- A pass that runs after region inference and builds new nodes must also copy `icnf-regions`, or the backends treat the site as unannotated (level 0).

## See Also

- [cg-kind-of](cg-kind-of.md)
- [pass-copy-spans](pass-copy-spans.md)
- [icnf-tree-structure](icnf-tree-structure.md)
