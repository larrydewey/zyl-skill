# boot-assemble-bundle

> Know what `selfhost/assemble.py` does to the compiler source: it strips `use` lines and non-driver `main`s, drops duplicate `defn`s (first wins), and flattens everything into one namespace.

## Why It Matters

Every boot stage compiles `selfhost/zyl_selfhost_compiler.zyl`, the concatenation of (roughly) `stdlib/core/*`, collections, allocator, the parts of `stdlib/math` the index's Ed25519 verification needs (words, bits, secret, bignum, sha512, ed25519), the compiler modules, `stdlib/lsp/builtins.zyl`, `stdlib/repl/*`, then `selfhost/driver.zyl` — the list in `assemble.py` is authoritative. The transformations hide bugs that appear when the same file is resolved standalone:

- `(use ...)` lines are removed: a missing `use` is invisible in the bundle ([pkg-use-what-you-construct](pkg-use-what-you-construct.md)).
- Every `main` except the driver's is stripped (`strip_named_defn`): a library `main` works in the bundle and breaks a real `use` ([pkg-library-no-main](pkg-library-no-main.md)).
- Duplicate `defn`s are **dropped, first occurrence wins**: a second definition of a name in a later file silently never runs.
- Files are re-rendered one paren per line to verify whole-bundle depth, then whitespace is collapsed to one ~680 KB line. Per-file deficits that cancel out are not caught ([boot-parens-per-file](boot-parens-per-file.md)).
- `contract_injection.zyl` is deliberately not in the bundle.

## Adding a module

Add it to `assemble.py`'s file list **before** `pipeline.zyl`, add its `(use compiler/...)` in `pipeline.zyl`, re-bundle and reseed.

## See Also

- [pass-adding-a-pass](pass-adding-a-pass.md)
- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
