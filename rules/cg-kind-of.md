# cg-kind-of

> Remember codegen decides print format, float arithmetic and string/variant equality from a static `kind-of` (0 word, 1 String, 2 Float, 3 variant) with no return-type inference.

## Why It Matters

This single mechanism explains a whole family of user-visible bugs: printing addresses for strings, float bits as integers, address comparison of strings. `kind-of` knows literals, annotated parameters and a few fixed-shape bodies; everything else defaults to 0 (word). Function result kinds are read off the body only when it has a fixed shape. `print` recognizes a few runtime string functions by name.

| Kind known | Effect |
|---|---|
| String | `print` uses `%s`; `=`/`!=` call `zyl_cstr_eq` |
| Float | `print` `%f`; SSE arithmetic; `comisd` |
| variant | `=`/`!=` → `zyl_variant_eq`; `<` etc. → `zyl_variant_cmp` |
| word (default) | integer ops, `%lld`, pointer equality |

## Improving it

Better kind propagation (e.g. from type inference results or function return shapes) would fix many user-level traps, but changes compiler output: reseed and run the interpreter differential category.

## See Also

- [fn-annotate-string-float-params](fn-annotate-string-float-params.md)
- [type-polymorphic-results-typed-printers](type-polymorphic-results-typed-printers.md)
