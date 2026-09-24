# boot-one-deftype-per-name

> Define each type name exactly once across everything one program imports, and keep variant names unique.

## Why It Matters

Functions are qualified per module, but constructor identity is not safe across modules. Duplicate `deftype`s create incompatible constructor identities: a constructor resolves to the declaration from the later `use`, and a match in the other module reads its tag with the other layout. Verified 2026-09-24: `Shape` declared as `(Circle Int) (Square Int)` in one module and `(Square Int) (Circle Int)` in another; `(Circle 2)` built in `main` was matched as `Square` by the first module (printed 8, not 4). Exhaustiveness checking also skips matches using shared variant names, and the VTable (`ast.zyl`) lets a later `deftype` reusing a variant name shadow the earlier one.

## Good

- Prefix variants by domain (`Tk*` tokens, `A*`/`Ast*` AST, `E*` ExprInner, `I*` ICNF, `CGS`/`CGE`/`CGR`/`CGP` codegen state).
- Grep before adding a type: `grep -rn "(deftype Name" stdlib/ selfhost/`.
- Prefer a small local type over a cross-module shared one when the dependency would point the wrong way.

## See Also

- [data-unique-variant-names](data-unique-variant-names.md)
