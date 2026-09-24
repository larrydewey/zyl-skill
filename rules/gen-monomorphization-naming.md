# gen-monomorphization-naming

> Know the per-type function names: impl methods are `Trait.method_Type`, and a trait-generic function's instances are `<key>~T1,T2` (argument types in order, fully spelled), at most 32 per function.

## Why It Matters

Spec §6.4 names specializations `fn_Type1_Type2` with sorted type names. The implementation instead names an instance after the function's canonical key plus `~` and its argument types in declaration order (`smaller~String,String`, `Desc.desc_...Vec~...Vec<Int>`): distinct type tuples always get distinct names, and the order is deterministic. Impl bodies are lifted by `monomorphization.zyl` as `Trait.method_Type` (type = the impl's type key, e.g. `Show.show_Int`).

## Notes

- Final linker symbols are mangled canonical keys (`zyl_mangle_key`, injective), e.g. `zy_local_x2Fmain_0__shapes__Area_x2Earea_...Rect`.
- `ZYL_DEBUG_TYPES=1` lists instances as they are made.
- Beyond 32 instances of one function, further calls use the shared body.
- The compiler itself contains instances of its own generic helpers, so a change to naming changes the fixed point.

## See Also

- [gen-per-type-instances](gen-per-type-instances.md)
- [pkg-canonical-keys](pkg-canonical-keys.md)
- [pass-monomorphization-tables](pass-monomorphization-tables.md)
