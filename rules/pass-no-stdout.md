# pass-no-stdout

> Never write to stdout from a compiler pass; emit warnings on stderr.

## Why It Matters

The same passes run inside `zyl-lsp`, where stdout is the JSON-RPC channel: a stray print corrupts the protocol stream. `unused_check` writes its `W_` warnings to stderr for this reason (which is also why the LSP cannot show them yet).

## See Also

- [tool-lsp-and-editors](tool-lsp-and-editors.md)
