# pass-adding-a-pass

> Add a compiler pass by writing the module, calling and `use`-ing it from `pipeline.zyl`, testing it, and reseeding.

## Steps

1. Write `stdlib/compiler/<name>.zyl` with a `use` for every module whose functions or constructors it touches; no `main`.
2. Call it at the right point in `stdlib/compiler/pipeline.zyl` and add its `(use compiler/<name>)` there. That `use` is all it takes for the compiler build to include it. The pipeline is shared by the CLI, `zyl eval` and the REPL.
3. Tests: `tests/regression/*.zyl` driving the pass directly; `tests/compile-fail/*.zyl` per error it raises.
4. `./boot.sh --bootstrap-from-self && ./boot.sh && ./run_regression_tests.sh --full`.

## Pipeline (as implemented)

```lisp
;; compile-to-exprs
(compile-check-balance srcbuf srcpath)             ; sexp_balance
(zyl-parse-file arena srcbuf srcpath)              ; lexer + parser -> Ast
(mr-resolve-program-full arena prog srcpath)       ; modules, qualify, orphan rule, -> Expr
(me-expand-program ...)                            ; macros
(compile-run-checks exprs resolved)                ; cc- dc- ac- mc- ec- uc- sc-
;; lower-exprs
(collect-definitions (inferer-new) exprs)          ; type inference
(monomorphize mono-ctx exprs)
(td-expand-program ...)                            ; trait dispatch
;; lower-after-mono: ci- (identity) -> al- (asserts) -> ic-program -> opt-optimize-fns -> ri-transform-fns
;; compile-to-fns stops at ICNF; compile-to-asm adds cg-program
```

## See Also

- [pass-copy-spans](pass-copy-spans.md)
- [pass-total-structural-match](pass-total-structural-match.md)
- [boot-module-build](boot-module-build.md)
