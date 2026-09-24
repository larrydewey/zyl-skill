# gen-monomorphization-naming

> Know that specialization names are canonical (sorted type names) and that the only per-type copies are impl methods named `Trait.method_Type`.

## Why It Matters

Spec §6.4/§17 name specializations `fn_Type1_Type2` with type names sorted alphabetically, so argument order does not matter and output is deterministic. The implementation (`canonical-name-from-type-map`) dedups, sorts and joins with `_`. Two known collisions: `f<Int,Int>` and `f<Int>` get the same name, and compound types are named by their outer constructor only (`f<List<Int>>` = `f<List<String>>`). Since user functions are compiled once, this mostly matters for impl methods and for compiler work: `type-to-string` output is part of the fixed point.

## Notes

- Final linker symbols are mangled canonical keys: `Area.area_Rect` in `shapes.zyl` becomes something like `zy_local_x2Fmain_0__shapes__Area_x2Earea_...Rect`.
- A bounded type parameter (unreachable from source today) would get one instantiation: the first type satisfying the bound.
- `check-trait-bound` accepts every primitive type for every trait.

## See Also

- [pkg-canonical-keys](pkg-canonical-keys.md)
- [pass-monomorphization-tables](pass-monomorphization-tables.md)
