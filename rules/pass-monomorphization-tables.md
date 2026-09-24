# pass-monomorphization-tables

> Treat `type-to-string` output as part of the fixed point: specialized symbol names are built from it.

## Why It Matters

`monomorphization.zyl` keeps its tables in one `MonoCtx` record (`MCGenFns`, `MCKnownFns`, `MCReturns`, `MCKTypes`, `MCStructs`, `MCAdtDefs`, `MCAdtInsts`, `MCTImpls`, `MCAdtParamOrder`), seeded from the `TypeInferer` by `mono-context-new`. `(monomorphize ctx exprs)` replaces generic functions by specializations for observed argument types, named via `canonical-name-from-type-map` over `type-to-string`. Changing how a type prints changes every specialized symbol in the compiler's own output → reseed.

## Notes

- Parameters starting with an uppercase letter are treated as type parameters.
- Impl bodies are lifted to `Trait.method_Type`.

## See Also

- [gen-monomorphization-naming](gen-monomorphization-naming.md)
