# pkg-modules-and-use

> A module is a `.zyl` file named by its path; import with `(use path)`, `(use pkg)`, `(use pkg:module)`, optionally with a whitespace-separated `{ ... }` list.

## Why It Matters

Module identity comes from the file path relative to its package root (without `.zyl`). A package's root module is named by the package name's last segment (`acme/greet` → `greet.zyl`). `(module name)` is accepted and ignored. The resolver splices the whole `use` graph into one compilation unit (whole-program; no separate compilation, no ABI).

## Forms

```lisp
(use collections/vec)                    ; module of this package, else stdlib, else dep root
(use acme/greet)                         ; a dependency's root module (same as `*`)
(use acme/greet *)
(use acme/greet { greet greet-twice })   ; checked import list
(use acme/greet { greet => hi })         ; local rename
(use acme/greet:text/shout { shout })    ; non-root module of a dependency
```

## Rules

- Single-segment lookup order is fixed: this package's module, then stdlib, then a dependency root.
- Package names contain `/`; `:` separates package and module, so forms never collide.
- A package not in `zyl.pkg` is `E_PKG_UNDECLARED_DEP`; a private symbol in a list `E_PKG_PRIVATE_SYMBOL`; a nonexistent one `E_PKG_UNKNOWN_SYMBOL`. Only imports from a **dependency** are checked: a list on a module of your own package (or in a lone file) is not validated, and every definition of the module is visible regardless (`(use util { nothere })` compiles).
- Cycles: within one package `E_MODULE_CYCLE`, across packages `E_PKG_CYCLE`.
- Missing file: lone file `E_MODULE_NOT_FOUND`; in a dependency `E_PKG_UNKNOWN_MODULE`; in the current package (currently) `E_PKG_UNDECLARED_DEP`.
- `unsafe` is a reserved module name (`E_PKG_RESERVED_MODULE`); `:unsafe` imports are parsed and ignored.
- The stdlib is one fully visible surface: `use`ing any stdlib module exposes every loaded stdlib definition. `core/core` is implicit.
- No commas in import lists: `,` is the reader's unquote prefix, so `{ greet => hi, greet-twice }` reads `, greet-twice` as an unquote form. It is a located `E_MALFORMED_FORM` ("an import list holds names only"; a trailing comma, `{ aa, }`: "`,` needs a form after it"). Before 2026-09-25 a trailing comma swallowed the rest of the file.

## See Also

- [pkg-library-no-main](pkg-library-no-main.md)
- [pkg-use-what-you-construct](pkg-use-what-you-construct.md)
- [pkg-pub-and-explicit-imports](pkg-pub-and-explicit-imports.md)
