# pass-state-threading

> Thread immutable state records through passes and return small wrapper ADTs when a function yields a value plus new state.

## Why It Matters

There is no shared mutable compiler state; passes communicate only through ADT values. This keeps passes deterministic and phase-isolated, and it is also the style the seed compiles most reliably.

```lisp
(deftype CGState (CGS Int Int Int Int (List REntry) (List FnName)))
;; arena, text buffer, next label, next slot, rodata, known functions
(deftype CGR (CGR CGState Int String))    ; state + label or slot
(defn cg-label-new (st) ...)              ; -> CGR with advanced state
```

## Notes

- Reconstruct records with fields in declaration order ([data-reconstruct-field-order](data-reconstruct-field-order.md)).
- Other records: `TypeInferer` (20 fields, accessors `ti-env`, `ti-subst`...), `MonoCtx`, `CGE`, `CGP`.
- Naming: lowering `ic-`, codegen `cg-`, module resolver `mr-`, balance `sb-`, macros `me-`, trait dispatch `td-`, closure inline `ci-`, assert lowering `al-`, optimizer `opt-`, regions `ri-`, checks `cc-` `dc-` `ac-` `mc-` `ec-` `uc-` `sc-`; env chain `EnvBind`/`EnvNil`; tokens `Tk*`; AST `A*`.

## See Also

- [pass-total-structural-match](pass-total-structural-match.md)
