# tool-lsp-and-editors

> Point any LSP client at `zyl-lsp` for the compiler's own diagnostics, type errors included; expect the pre-type checks one at a time, every type error at once, no capability check, byte-based columns and name-based (not scope-based) navigation.

## Why It Matters

`zyl-lsp` is built from the same compiler and runs the same front end in the same order (`stdlib/lsp/document_manager.zyl`): parse under the document's own path, resolve modules, expand macros, then the checks (duplicates, arity, mutability, exhaustiveness, Secret), then derive expansion, impl lifting, closure inlining and the **type checker** (`dm-type-diagnostics`). So its errors are the compiler's errors: balance with quick-fix, the checks, and since 2026-09-25 (`f4213bc`) every type error located in the document, which are the main class of error under sound typing. `unused_check`'s unused-binding and shadowing warnings are published as Warning diagnostics. Each diagnostic is placed at the `--> file:line:col` the compiler reports and shows the headline plus the `help` line (not the rendered excerpt). Navigation positions come from a text scan of the document; meanings from the real front end; the two meet by name.

## Setup

- VS Code: `./install.sh --with-vscode` (needs `npm`); extension 0.4.0 is esbuild-bundled (`npx vsce package` → `zyl-0.4.0.vsix`); finds the server via `zyl.lsp.path`, `$ZYL_HOME/bin`, `~/.zyl/bin`, a workspace's `build/boot`, then `PATH`. **Zyl: Run Current File** = `Ctrl+Shift+Enter`.
- VS Code tasks: the extension contributes a `$zyl` problem matcher (headline `error[CODE]: msg` / `warning[CODE]: msg` + `--> file:line:col`), used by its own file and package tasks; use it in your `tasks.json` with `"problemMatcher": "$zyl"`.
- Neovim: `vim.lsp.config.zyl = { cmd = { vim.fn.expand('~/.zyl/bin/zyl-lsp') }, filetypes = { 'zyl' } }`.
- Emacs eglot / Helix: point at `~/.zyl/bin/zyl-lsp`, associate `.zyl`.
- `./boot.sh` builds `build/boot/zyl-lsp` and refreshes an installed `~/.zyl/bin/zyl-lsp`; an editor running an old server shows old diagnostics until it restarts the server.

## Limits

- The checks before the type checker stop at their first problem, so those errors come one at a time, and a document that fails one of them gets no type errors until it is fixed. The type checker reports every type error at once; warnings are reported all together.
- `capability_check` is not run: a package capability violation shows up only when `zyl` compiles.
- Nothing after type checking runs (no ICNF, no region inference), so `E_REGION_ESCAPE` and other back-end errors are not shown.
- Positions are **byte** columns, not UTF-16 code units: on a line with a non-ASCII character before it, a diagnostic or hover column is off. UTF-8 text itself survives both directions (and `\uXXXX` escapes, surrogate pairs included, decode to UTF-8).
- Completion has no local variables; hover shows a parameter's **declared** annotation by its source name (e.g. `StrView`), not an inferred type.
- Rename is textual within the file: it also renames a shadowing local.
- Navigation covers open documents only.
- Memory: the JSON codec is linear (one growable buffer per message), but the front end and type checker allocate on the process heap, which is never freed: opening a 150 KB document takes about 470 MB, and its semantic tokens about 500 MB more.

## Notes

- A compiler pass that also runs in the LSP must never write to stdout (the JSON-RPC channel); use stderr. Passes that report through `zyl_warn_emit` are captured (`zyl_warn_capture`) and turned into diagnostics; a pass that panics becomes one diagnostic instead of killing the server.
- `tests/lsp/lsp_protocol_test.py` drives the real binary (108 checks, including a type error, one diagnostic per type error, a 150 KB document and UTF-8).

## See Also

- [pass-no-stdout](pass-no-stdout.md)
