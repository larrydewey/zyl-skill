# tool-lsp-and-editors

> Point any LSP client at `zyl-lsp` for the compiler's own diagnostics; expect one error at a time (unused/shadowing warnings all together), no type inference or capability check, and name-based (not scope-based) navigation.

## Why It Matters

`zyl-lsp` is built from the same compiler, so its errors are the compiler's errors (balance with quick-fix, duplicates, arity, mutability, exhaustiveness, Secret), and `unused_check`'s unused-binding and shadowing warnings are published as Warning diagnostics. Each diagnostic is placed at the `--> file:line:col` the compiler reports and shows the headline plus the `help` line (not the rendered excerpt). Navigation positions come from a text scan of the document; meanings from the real front end (parse, resolve, expand; no type inference); the two meet by name.

## Setup

- VS Code: `./install.sh --with-vscode` (needs `npm`); extension 0.4.0 is esbuild-bundled (`npx vsce package` → `zyl-0.4.0.vsix`); finds the server via `zyl.lsp.path`, `$ZYL_HOME/bin`, `~/.zyl/bin`, a workspace's `build/boot`, then `PATH`. **Zyl: Run Current File** = `Ctrl+Shift+Enter`.
- VS Code tasks: the extension contributes a `$zyl` problem matcher (headline `error[CODE]: msg` / `warning[CODE]: msg` + `--> file:line:col`), used by its own file and package tasks; use it in your `tasks.json` with `"problemMatcher": "$zyl"`.
- Neovim: `vim.lsp.config.zyl = { cmd = { vim.fn.expand('~/.zyl/bin/zyl-lsp') }, filetypes = { 'zyl' } }`.
- Emacs eglot / Helix: point at `~/.zyl/bin/zyl-lsp`, associate `.zyl`.

## Limits

- One error at a time (checks stop at the first problem); warnings are reported all together.
- `capability_check` not run; no type inference (so no type errors).
- Completion has no local variables; hover shows declared, not inferred, types.
- Rename is textual within the file: it also renames a shadowing local.
- Navigation covers open documents only.

## Notes

- A compiler pass that also runs in the LSP must never write to stdout (the JSON-RPC channel); use stderr.
- `tests/lsp/lsp_protocol_test.py` drives the real binary.

## See Also

- [pass-no-stdout](pass-no-stdout.md)
