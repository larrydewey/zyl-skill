# icnf-new-form-needs-case

> Give every new special form its own case in `ic-expr-node`; lowering turns anything unrecognized into `(IConst 0)` silently.

## Why It Matters

The fail-soft default compiles wrong programs quietly. `for`, `spawn` and `with-resource` all once lowered to 0 this way, and today `assert`, `unwrap`, `read-line`, `exit`, `close`, `make-struct`, `make-variant` still do. A new form that parses and passes every check but lacks a lowering case evaluates to 0 with no diagnostic.

## Checklist for a new form

1. Recognize it in `convert-ast` / `dispatch-special` (`expr_inner.zyl`) — the single recognition point (no-dispatch parsing).
2. Walk it in **every** pass that rewrites `ExprInner` (macro expansion, checks, monomorphization, trait dispatch, closure inline, assert lowering) — rewriters list every constructor.
3. Lower it in `ic-expr-node` (`icnf.zyl`).
4. Handle it in the optimizer, region inference (`ri-name-safe-in`), codegen and the interpreter if it introduces a new `Icnf` node.
5. Add it to `stdlib/lsp/builtins.zyl` for hover/completion.
6. Test compiled **and** `zyl eval` output (interpreter category).

## See Also

- [pass-total-structural-match](pass-total-structural-match.md)
- [fn-unlowered-forms](fn-unlowered-forms.md)
