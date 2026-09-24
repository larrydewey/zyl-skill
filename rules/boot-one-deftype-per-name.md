# boot-one-deftype-per-name

> Define each type name exactly once across the whole bundle, and keep variant names unique.

## Why It Matters

In the concatenated bundle everything is one flat namespace. Duplicate `deftype`s create incompatible constructor identities: construction uses the later declaration, pattern matches against the other silently fail, and exhaustiveness checking skips matches using shared variant names. The VTable (`ast.zyl`) lets a later `deftype` reusing a variant name shadow the earlier one.

## Good

- Prefix variants by domain (`Tk*` tokens, `A*`/`Ast*` AST, `E*` ExprInner, `I*` ICNF, `CGS`/`CGE`/`CGR`/`CGP` codegen state).
- Grep before adding a type: `grep -rn "(deftype Name" stdlib/ selfhost/`.
- Prefer a small local type over a cross-module shared one when the dependency would point the wrong way.

## See Also

- [data-unique-variant-names](data-unique-variant-names.md)
