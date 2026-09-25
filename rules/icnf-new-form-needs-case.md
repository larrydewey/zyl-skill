# icnf-new-form-needs-case

> Give every new special form a case in the type pass and in `ic-expr-node`, and every new `Icnf` node a case in each ICNF pass and in both backends: lowering and the native backend's `ml-expr` turn anything they do not recognize into 0 silently.

## Why It Matters

The front end now fails closed: a form whose parser rejected its shape is `E_MALFORMED_FORM` (`arity_check.zyl`, from its `EUnknown`), and a form the type pass has no case for is `E_CANNOT_INFER: no type for form not typed` (`ta-expr-node`'s catch-all). But a form that parses **and** type-checks and has no lowering case still becomes `(IConst 0)` in `ic-expr-node` with no diagnostic. `for`, `spawn`, `with-resource`, `assert` and `unwrap` all once lowered to 0 this way. Verified 2026-09-25: `read-line`, `exit` and `close` still do (`(exit 3)` exits with status 0; `(print (read-line))` prints `(null)`), while `make-struct` in expression position is `E_CANNOT_INFER` and a bad `make-variant` is `E_MALFORMED_FORM`.

The native backend has the same trap one level down: `ml-expr`'s catch-all is `(ml-const ms 0)`, so a node `ml-ok` accepts but `ml-expr` does not lower computes 0.

## Checklist for a new form

1. Recognize it in `convert-ast` / `dispatch-special` (`expr_inner.zyl`) — the single recognition point (no-dispatch parsing) — and give a bad shape an `EUnknown` so it is `E_MALFORMED_FORM`.
2. Walk it in **every** pass that rewrites or reads `ExprInner`: macro expansion, qualification, the checks (`cc-` `dc-` `ac-` `mc-` `ec-` `uc-` `sc-`), derive, impl lifting, closure inlining, and the type pass (`ta-expr-node` for its type, `ta-copy-node` so specialized instances copy it).
3. Lower it in `ic-expr-node` (`icnf.zyl`).
4. If it introduces a new `Icnf` node: `icnf-has-set` and `icnf-sets` (evaluation order), the optimizer and inliner (`opt-expr-node`, `opt-size`, `opt-plain`, `opt-rename`, `opt-subst`...), region inference (`ri-name-safe-in` and `rg-expr-node`, both total matches, so a missing case is `E_NON_EXHAUSTIVE_MATCH`), the reuse pass (`ru-walk-node`, `ru-occurs`, `ru-calls-any`, `ru-opaque-refs`), `icnf_print`, the stack machine (`cg-expr`, `kind-of-base`, `icnf-size` and slot counting), the interpreter (`stdlib/repl/interp.zyl`), and either `ml-ok` returns false for it or `ml-ok`, `ml-expr` (and `ml-tail`) handle it together.
5. Add it to `stdlib/lsp/builtins.zyl` for hover/completion.
6. Test compiled (with and without `ZYL_MIR=0`) **and** `zyl eval` output (interpreter category).

## See Also

- [pass-total-structural-match](pass-total-structural-match.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
- [fn-unlowered-forms](fn-unlowered-forms.md)
