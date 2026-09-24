# cg-kind-of

> Remember codegen decides print format, float arithmetic and string/variant comparison from `kind-of` (0 word, 1 String, 2 Float, 3 variant): the legacy literal/annotation kind first, else the type-annotation kind stored per ICNF node (attr table 1).

## Why It Matters

`kind-of-base` knows literals, annotated parameters, a few fixed-shape bodies and string-returning FFI symbols. When it answers 0, `kind-of` reads the node's kind from attr table 1, which ICNF lowering filled from `ta-kind` (the inferred type: String 1, Float 2). Only 1 and 2 come from inference; kind 3 is still only for constructor expressions. A conflicting or unknown type gives 0, the old word behavior.

| Kind known | Effect |
|---|---|
| String | `print` uses `%s`; `=`/`!=` call `zyl_cstr_eq`; `<`/`>`/`<=`/`>=` call `zyl_cstr_cmp` |
| Float | `print` `%f`; SSE arithmetic; `comisd` |
| variant | `=`/`!=` → `zyl_variant_eq` (fallback only: ADT `==` of a known type usually became a call to `T.==` in the type pass); `<` etc. → `zyl_variant_cmp` |
| word (default) | integer ops, `%lld`, pointer equality |

## Keeping kinds

A pass that rebuilds ICNF nodes must carry the kind across (`ic-keep-kind`, as `ic-hoist`, the optimizer and region inference do): see [pass-keep-kinds](pass-keep-kinds.md). `print` of a value with a `Show` impl is lowered to a `Show.show` call (attr table 3) before codegen sees it.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [gen-per-type-instances](gen-per-type-instances.md)
