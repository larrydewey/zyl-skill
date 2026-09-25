# pass-adding-a-pass

> Add a compiler pass by writing the module, calling and `use`-ing it from `pipeline.zyl` at the right point in the phase order, testing it, and reseeding.

## Steps

1. Write `stdlib/compiler/<name>.zyl` with a `use` for every module whose functions or constructors it touches; no `main`. It must type-check under the sound checker like everything else.
2. Call it at the right point in `stdlib/compiler/pipeline.zyl` and add its `(use compiler/<name>)` there. That `use` is all it takes for the compiler build to include it. The pipeline is shared by the CLI, `zyl eval` and the REPL. Give a transformation an off switch (`ZYL_<NAME>=0`, read with `zyl_getenv_str`) like the existing ones, so a miscompile can be bisected.
3. Tests: `tests/regression/*.zyl` driving the pass directly; `tests/compile-fail/*.zyl` per error it raises (with its `; expect-error: CODE` line).
4. `./boot.sh --bootstrap-from-self && ./boot.sh && ./run_regression_tests.sh --full`.

## Pipeline (as implemented)

```lisp
;; compile-to-exprs
(field-types-clear)
(compile-check-balance srcbuf srcpath)             ; sexp_balance
(zyl-parse-file arena srcbuf srcpath)              ; lexer + parser -> Ast
(mr-resolve-program-full arena prog srcpath)       ; modules, qualify (list literals, quote,
                                                   ;   quasiquote rewritten here), orphan rule -> Expr
(me-expand-program ...)                            ; macros
(compile-run-checks exprs resolved)                ; cc- dc- ac- mc- ec- uc- sc-
;; lower-exprs
(dv-expand-program exprs0)                         ; derive -> impl blocks
(lift-impls exprs)                                 ; impl bodies -> Trait.method_Type functions
;; lower-after-mono
(ci-expand-program arena td-exprs)                 ; closure inlining (identity today)
(ta-annotate ci-exprs)                             ; sound HM: every type error, then fail;
                                                   ;   trait resolution, per-type instances f~T
(ic-program arena ...)                             ; -> ICNF
(opt-optimize-fns (opt-inline-fns fns))            ; inline + copy propagation, then fold
(rg-regions (ri-transform-fns opt-fns))            ; stack variants, region placement
(ru-reuse reg-fns)                                 ; in-place reuse, owning clones f~own
;; compile-to-fns stops here; compile-to-asm adds codegen-fns:
(cg-program-file (cg-new arena) region-fns srcpath) ; per function: native backend or stack machine
```

## Where a new pass goes

- A check over `Expr`: in `compile-run-checks`, after macro expansion (so macro output is checked) and before the type pass.
- An `Expr` rewrite: before `ta-annotate`; after it, rebuilding nodes loses their types ([pass-keep-kinds](pass-keep-kinds.md)).
- An ICNF rewrite: between `ic-program` and `rg-regions` if region inference should see its output (as inlining does); after `rg-regions` only if it keeps every node's `icnf-regions` entry and reads the summaries rather than changing escape (as the reuse pass does).
- Anything per function below ICNF belongs in a backend ([cg-native-backend-mir](cg-native-backend-mir.md)).

## See Also

- [pass-copy-spans](pass-copy-spans.md)
- [pass-keep-kinds](pass-keep-kinds.md)
- [pass-total-structural-match](pass-total-structural-match.md)
- [icnf-optimizer-scope](icnf-optimizer-scope.md)
- [boot-module-build](boot-module-build.md)
