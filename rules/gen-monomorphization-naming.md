# gen-monomorphization-naming

> Know the per-type function names: impl methods are `Trait.method_<type key>`, and a trait-generic function's instances are `<key>~T1,T2` (argument types in order, fully spelled); more than 256 instances of one function is `E_CANNOT_INFER`.

## Why It Matters

Spec §6.4 names specializations `fn_Type1_Type2` with sorted type names. The implementation instead names an instance after the function's canonical key plus `~` and its argument types in declaration order: `local/main@0::p::smaller~String,String`, `show-it~zyl/std@5::core/option::Option<String>`. Distinct type tuples always get distinct names, and the order is deterministic. Impl bodies are lifted by `lift_impls.zyl` as `Trait.method_<type key>`: `Show.show_Int` for a primitive, `local/main@0::shapes::Area.area_local/main@0::shapes::Rect` for a program type. The generic original of a trait-generic function is dropped from the program; only its instances are emitted.

## Notes

- Final linker symbols are mangled canonical keys (`zyl_mangle_key`, injective), e.g. `zy_local_x2Fmain_0__mn__smaller_x7EString_x2CString`.
- `ZYL_DEBUG_TYPES=1` lists every function's type, instances included, as `name : (Args -> Result)`.
- A function that would need more than 256 instances (polymorphic recursion, usually) is `E_CANNOT_INFER: ... would need more than 256 specialized instances`. There is no fallback to a shared body.
- The compiler itself contains instances of its own generic helpers, so a change to naming changes the fixed point.

## See Also

- [gen-per-type-instances](gen-per-type-instances.md)
- [pkg-canonical-keys](pkg-canonical-keys.md)
- [pass-instance-naming](pass-instance-naming.md)
