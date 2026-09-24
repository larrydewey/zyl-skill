# pass-copy-spans

> When a pass rebuilds an `Expr` node, copy the original's source span onto the replacement: `(ffi-call "zyl_span_copy" new-node old-node 1000)`.

## Why It Matters

Nodes carry no span field. The reader records each node's byte offset in a runtime table keyed by node address. That is how a late error such as codegen's `E_UNBOUND_VARIABLE` still prints `--> file:line:col` with a caret. A pass that rewrites nodes and skips the copy loses locations for everything downstream. (The table is only ever probed by key, never iterated, so address keys don't break determinism.)

## See Also

- [pass-diagnostics](pass-diagnostics.md)
