# tool-lsp-and-editors

> Point any LSP client at `zyl-lsp` for the compiler's own diagnostics; expect one diagnostic at a time, no `W_` warnings, no capability check, and name-based (not scope-based) navigation.

## Why It Matters

`zyl-lsp` is built from the same compiler, so its errors are the compiler's errors (balance with quick-fix, duplicates, arity, mutability, exhaustiveness, Secret). Positions come from a text scan of the document; meanings from the real front end (parse, resolve, expand; no type inference); the two meet by name.

## Setup

- VS Code: `./install.sh --with-vscode` (needs `npm`); finds the server via `zyl.lsp.path`, `$ZYL_HOME/bin`, `~/.zyl/bin`, a workspace's `build/boot`, then `PATH`. **Zyl: Run Current File** = `Ctrl+Shift+Enter`.
- Neovim: `vim.lsp.config.zyl = { cmd = { vim.fn.expand('~/.zyl/bin/zyl-lsp') }, filetypes = { 'zyl' } }`.
- Emacs eglot / Helix: point at `~/.zyl/bin/zyl-lsp`, associate `.zyl`.

## Limits

- One diagnostic at a time (checks stop at the first problem).
- `unused_check` and `capability_check` not run.
- Completion has no local variables; hover shows declared, not inferred, types.
- Rename is textual within the file: it also renames a shadowing local.
- Navigation covers open documents only.

## Notes

- A compiler pass that also runs in the LSP must never write to stdout (the JSON-RPC channel); use stderr.
- `tests/lsp/lsp_protocol_test.py` drives the real binary.

## See Also

- [pass-no-stdout](pass-no-stdout.md)
