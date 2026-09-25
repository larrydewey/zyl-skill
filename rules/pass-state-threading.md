# pass-state-threading

> Thread state records through a pass and return small wrapper ADTs when a function yields a value plus new state; when a pass needs a table keyed by node or name instead, use a typed runtime table that is cleared at the start of each program and only probed by key.

## Why It Matters

Passes communicate through ADT values and through a fixed set of side tables. Keeping everything else local keeps passes deterministic and phase-isolated, and the REPL and the language server, which run the pipeline many times in one process, correct.

Threaded records, as in codegen and the native backend's lowering:

```lisp
(deftype CGState (CGS Arena StrBuf Int Int (List REntry) (List FnName)))
;; arena, text buffer, next label, next slot, rodata, known functions
(deftype CGR (CGR CGState Int String))    ; state + label or slot
(defn cg-label-new (st) ...)              ; -> CGR with advanced state

(deftype MS (MS Int Int (List MI) (List Int)))  ; next vreg, next label, instructions, blocks
(deftype MR (MR MS Int))                        ; state + vreg
```

Keyed tables, where a record would be threaded through every node for nothing: `node_tables.zyl`'s `zyl_attrh_*` tables on nodes, `zyl_smap_*` maps by name (`secret-marks`, `ru-facts`, `rg-summaries`, the type pass's tables in `TaSt`), `zyl_ref` cells for a walk's running state (`ru-taint-cell`, `ml-view-data`), and per-function scratch arenas reset between functions (`mir-arena`). Each is cleared where its program or function starts (`node-tables-clear`, `ta-annotate-program`, `ru-reuse`, `mir-reset`, `cg-new`).

## Notes

- Reconstruct records with fields in declaration order ([data-reconstruct-field-order](data-reconstruct-field-order.md)).
- A table may be probed by key, never iterated: iteration order of a hash table is not deterministic ([det-no-address-dependent-output](det-no-address-dependent-output.md)). Order comes from lists and arrays.
- A table that is not cleared leaks one program's facts into the next REPL entry or LSP analysis (a node address can be reused).
- Naming: lowering `ic-`, codegen stack machine `cg-`, native backend lowering `ml-` and emission `mb-`, MIR and register allocation `mir-`, parallel moves `pm-`, module resolver `mr-`, qualification `qf-`, balance `sb-`, macros `me-`, type pass `ta-`, derive `dv-`, closure inline `ci-`, optimizer and inliner `opt-`, region inference `ri-`/`rg-`, reuse `ru-`, checks `cc-` `dc-` `ac-` `mc-` `ec-` `uc-` `sc-`, errors `err-`; env chain `EnvBind`/`EnvNil`; tokens `Tk*`; AST `A*`.

## See Also

- [pass-total-structural-match](pass-total-structural-match.md)
- [pass-keep-kinds](pass-keep-kinds.md)
