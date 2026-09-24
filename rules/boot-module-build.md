# boot-module-build

> The compiler is built like any program: every boot stage compiles `selfhost/driver.zyl`, and module resolution follows its `(use ...)` tree through `stdlib/`. A compiler module is reached only through a `use`.

## Why It Matters

Since 2026-09-24 there is no bundle. `selfhost/assemble.py` and `selfhost/zyl_selfhost_compiler.zyl` are gone, and with them every transformation that used to hide bugs (stripped `use` lines, stripped library `main`s, first-wins `defn` dedupe, one-line whitespace collapse, a whole-bundle depth check). What that means for compiler source:

- Each top-level name is qualified to its module (`zyl/std@5::compiler/lexer::lex-loop`), so two modules may define the same function name. A file that `use`s both sees the one from the **later** `use`.
- A missing `use` is an ordinary error: `E_UNBOUND_VARIABLE` at the call ([pkg-use-what-you-construct](pkg-use-what-you-construct.md)).
- A library `main` collides with the driver's: `E_DUPLICATE_DEFINITION` ([pkg-library-no-main](pkg-library-no-main.md)).
- Every file is balance-checked on its own before it is parsed ([boot-parens-per-file](boot-parens-per-file.md)).
- Type and variant names still clash across modules ([boot-one-deftype-per-name](boot-one-deftype-per-name.md)).
- Contracts are lowered in `expr_inner.zyl` (`contract-defn-body`, `contract-check`); there is no separate contract-injection module.

## Where the source comes from

`boot.sh` first copies `stdlib/` and the runtime into `build/boot/` and exports `ZYL_HOME=build/boot`, so every stage resolves this checkout's source, never an installed `~/.zyl`. The output does not depend on the checkout path or the working directory.

## Adding a module

Write it under `stdlib/compiler/`, `use` it from the module that calls it (a new pass: from `pipeline.zyl`), then reseed ([boot-fixed-point-workflow](boot-fixed-point-workflow.md)). No file list to edit.

## See Also

- [pass-adding-a-pass](pass-adding-a-pass.md)
- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
