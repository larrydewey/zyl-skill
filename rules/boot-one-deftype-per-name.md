# boot-one-deftype-per-name

> Define each type name exactly once across everything one program imports, and keep variant names unique.

## Why It Matters

Functions are qualified per module, but constructor names are not: a bare constructor resolves to the declaration from the later `use`, and the VTable (`ast.zyl`) lets a later `deftype` reusing a variant name shadow the earlier one. On 2026-09-24 that was a silent miscompile (`Shape` declared `(Circle Int) (Square Int)` in one module and `(Square Int) (Circle Int)` in another: `(Circle 2)` built in `main` was matched as `Square`, printing 8, not 4).

Since sound type checking (verified 2026-09-25) the same program is rejected, but with confusing errors:

- Two `Shape`s: `error[E_TYPE_MISMATCH]: cannot unify Shape with Shape` at every call that mixes them (the message prints both types by their short name).
- A variant name shared by two different types (`Leaf` in `T1` and `T2`): a constructor or pattern in a module that `use`s both means the later module's type, so `(t1v (Leaf 2))` is `cannot unify T1 with T2`, and a match on `Leaf` alone can be reported as a non-exhaustive match over the *other* type.

Programs that happen to use each constructor only in its own module still compile and run correctly, which makes the collision easy to miss until a third module imports both.

**Function names collide the same way, and worse.** The stdlib shipped two
types both called `Map` (`core/map`, generic; `collections/map`, an
arena-backed Int table) which also both exported `map-get`, `map-remove`
and `map-has` — with *incompatible* signatures, 2 arguments against 3. A
program that imported both was accepted, and the three shared names
resolved to the Int-only module, so `core/map`'s versions became
unreachable. The symptom was `E_ARITY_MISMATCH: map-get called with 2
argument(s), but it takes 3` on correct code — no mention of `Map` at all.

Note what that is and is not. The type checker did its job: the two `Map`s
never unify, and a value of the wrong one is caught. What it cannot catch is
the *name* resolving to a function that exists but is not the one you meant,
because both candidates are well-typed. A collision where the losing side
is not diagnosed is worse than one that fails loudly, so when a rename is
available, take it: the Int table is `IntMap` in `collections/intmap` now,
and the three shared names moved with it.

`duplicate_check.zyl` had claimed the pair was "never `use`d together" and
used that to justify checking names only within one resolved program. The
claim was an assumption about which modules those were, and this pair was
the counterexample — nothing enforced it. A name being reused across
modules is not by itself evidence the reuse is safe.

## Good

- Prefix variants by domain (`Tk*` tokens, `A*`/`Ast*` AST, `E*` ExprInner, `I*` ICNF, `CGS`/`CGE`/`CGR`/`CGP` codegen state).
- Grep before adding a type: `grep -rn "(deftype Name" stdlib/ selfhost/`.
- Prefer a small local type over a cross-module shared one when the dependency would point the wrong way.

## See Also

- [data-unique-variant-names](data-unique-variant-names.md)
