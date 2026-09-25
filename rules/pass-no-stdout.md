# pass-no-stdout

> Never write to stdout from a compiler pass; emit every warning and every reported error through `zyl_warn_emit`.

## Why It Matters

The same passes run inside `zyl-lsp`, where stdout is the JSON-RPC channel: a stray print corrupts the protocol stream. The runtime sink `zyl_warn_emit` writes to stderr normally; under `zyl_warn_capture` (the language server's `document_manager.zyl`) it appends to a buffer that the server parses into diagnostics. That is how the unused and shadowing warnings (`uc-warn-line`, `err-warn-at`) and every type error (`ta-type-error`) reach the editor. Writing to stderr directly (`file-write 2`, `fprintf(stderr)`) is safe for the protocol but bypasses the capture, so the editor never sees it.

## Notes

- `print` in a pass is also a stack-machine-only construct: it keeps that function out of the native backend.
- `ZYL_DEBUG_STAGES` tracing goes to `/tmp/dbg`, not stdout, for the same reason.
- `ZYL_REUSE_DEBUG` and `ZYL_DEBUG_TYPES` dumps go through `zyl_warn_emit` too.

## See Also

- [tool-lsp-and-editors](tool-lsp-and-editors.md)
- [pass-diagnostics](pass-diagnostics.md)
