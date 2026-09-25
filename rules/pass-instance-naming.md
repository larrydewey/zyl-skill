# pass-instance-naming

> Treat the type pass's canonical type text (`ta-canon`) as part of the fixed point: per-type instance names `f~T` are built from it, and there is no separate monomorphization pass any more.

## Why It Matters

`monomorphization.zyl` (with its `MonoCtx` tables) and `type_inference.zyl` are deleted. Specialization now happens inside the type pass (`type_annotate.zyl`): a trait-generic function (one whose body uses a trait method or a class such as `Num`), and a function value of one, is specialized at each use to an instance named `<name>~<args>` (`ta-specialize`), where `<args>` is `ta-canon-list` of the argument types: a nullary type is its name, an applied one `Name<A,B>`, a function `fn<A,B>R`, an unresolved variable `?`. The instance is a deep copy of the definition (`ta-copy-defn`, spans copied) appended to the program, and every call or value use is renamed to it (`node-calls`); the generic original is then dropped (`ta-drop-generic`), so an unresolved use fails as an undefined function instead of running a guessed impl.

```asm
call zy_local_x2Fmain_0__t8__sq_x7EInt                    ; sq~Int
call zy_local_x2Fmain_0__t8__add3_x7EInt_x2CInt_x2CInt    ; add3~Int,Int,Int
```

Changing how `ta-canon` prints a type, or which functions count as trait-generic, renames specialized symbols throughout the compiler's own output: reseed.

## Notes

- A function needing more than 256 instances is `E_CANNOT_INFER` (`would need more than 256 specialized instances`): polymorphic recursion whose argument types grow.
- Impl bodies are lifted to `Trait.method_Type` by `lift_impls.zyl` before the type pass.
- An annotated parameter (`(x Int)`) makes the function concrete; an unannotated one whose body uses `*` is `Num`-generic and gets an `~Int` (or `~Float`) instance per use type.
- The reuse pass adds another suffix, `f~own` ([icnf-reuse-pass](icnf-reuse-pass.md)); `cg-base-name` strips everything from the first `~` when it looks up a function's `Secret` marks.

## See Also

- [gen-monomorphization-naming](gen-monomorphization-naming.md)
- [gen-per-type-instances](gen-per-type-instances.md)
- [cg-symbols-and-entry](cg-symbols-and-entry.md)
