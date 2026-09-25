# cg-kind-of

> Remember codegen decides print format, float arithmetic and string/variant comparison from `kind-of` (0 word, 1 String, 2 Float, 3 variant): the literal/shape kind first, else the kind the type pass recorded for the ICNF node (`icnf-kinds`); and any non-word operator keeps a function out of the native backend.

## Why It Matters

`kind-of-base` knows literals, annotated parameters, a few fixed-shape bodies and string-returning runtime symbols. When it answers 0, `kind-of` reads the node's kind from the `icnf-kinds` side table (`node_tables.zyl`), which ICNF lowering filled from `ta-kind` (the inferred type: String 1, Float 2). Only 1 and 2 come from inference; kind 3 is still only for constructor expressions. Since type checking is sound, every String and Float operand has a known type, so the word fallback no longer hides a mistyped operand.

| Kind known | Effect |
|---|---|
| String | `print` uses `%s`; `=`/`!=` call `zyl_cstr_eq`; `<`/`>`/`<=`/`>=` call `zyl_cstr_cmp` |
| Float | `print` `%f`; SSE arithmetic; `comisd` |
| variant | `=`/`!=` → `zyl_variant_eq` (fallback only: ADT `==` of a known type became a call to the generated `T.==` in the type pass); `<` etc. → `zyl_variant_cmp` |
| word (default) | integer ops, `%lld`, pointer equality |

## The native backend

`ml-ok` accepts an `IBinop` only when `cg-int-operands` says neither operand has kind 1, 2 or 3, and it never accepts `IPrint`. So a function with a String, Float or ADT operator, or a `print`, is compiled whole by the stack machine ([cg-stack-machine-fallback](cg-stack-machine-fallback.md)). The native backend itself uses `kind-of` only to extend its environment.

## Keeping kinds

A pass that rebuilds ICNF nodes must carry the kind across (`ic-keep-kind`, or `opt-keep` which also carries the span and the scalar and ADT marks): see [pass-keep-kinds](pass-keep-kinds.md). `print` of a value with a `Show` impl is lowered to a `Show.show` call (the `node-shows` table) marked kind 1 before codegen sees it.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [gen-per-type-instances](gen-per-type-instances.md)
- [pass-keep-kinds](pass-keep-kinds.md)
