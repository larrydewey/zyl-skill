# boot-one-deftype-per-name

> Define each type name exactly once across everything one program imports, and keep variant names unique.

## Why It Matters

Functions are qualified per module, but constructor names are not: a bare constructor resolves to the declaration from the later `use`, and the VTable (`ast.zyl`) lets a later `deftype` reusing a variant name shadow the earlier one. On 2026-09-24 that was a silent miscompile (`Shape` declared `(Circle Int) (Square Int)` in one module and `(Square Int) (Circle Int)` in another: `(Circle 2)` built in `main` was matched as `Square`, printing 8, not 4).

Since sound type checking (verified 2026-09-25) the same program is rejected, but with confusing errors:

- Two `Shape`s: `error[E_TYPE_MISMATCH]: cannot unify Shape with Shape` at every call that mixes them (the message prints both types by their short name).
- A variant name shared by two different types (`Leaf` in `T1` and `T2`): a constructor or pattern in a module that `use`s both means the later module's type, so `(t1v (Leaf 2))` is `cannot unify T1 with T2`, and a match on `Leaf` alone can be reported as a non-exhaustive match over the *other* type.

Programs that happen to use each constructor only in its own module still compile and run correctly, which makes the collision easy to miss until a third module imports both.

## Good

- Prefix variants by domain (`Tk*` tokens, `A*`/`Ast*` AST, `E*` ExprInner, `I*` ICNF, `CGS`/`CGE`/`CGR`/`CGP` codegen state).
- Grep before adding a type: `grep -rn "(deftype Name" stdlib/ selfhost/`.
- Prefer a small local type over a cross-module shared one when the dependency would point the wrong way.

## See Also

- [data-unique-variant-names](data-unique-variant-names.md)
