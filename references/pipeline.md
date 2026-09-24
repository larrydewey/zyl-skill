# Compilation Pipeline

## As specified (§22, strict order, no back edges)

1 Parsing → 2 Macro expansion → 3 Type inference + trait resolution → 4 Region inference + capture analysis → 5 Monomorphization → 6 ICNF → 7 Optimization (safe only) → 8 Codegen → 9 Linking → 10 Contract injection (optional) → 11 Hash finalization. §31.9 puts capability enforcement after module resolution, before type inference.

## As implemented (`stdlib/compiler/pipeline.zyl`, `compile-to-asm`)

| # | Stage | Function / module | Can raise |
|---|---|---|---|
| 1 | Balance check | `compile-check-balance` / `sexp_balance.zyl` (`sb-check-string`, `sb-hint`) | `E_UNBALANCED_*`, `E_UNTERMINATED_STRING` |
| 2 | Lex + parse (no-dispatch: every form a generic list) | `zyl-lex`, `zyl-parse-file` / `lexer.zyl`, `parser.zyl`, `ast.zyl` | `E_MALFORMED_PARAMETER`, `E_BYTE_VALUE_OOB`, `E_MATCH_NONEXHAUSTIVE`, `E_RESERVED_KEYWORD`, field `set!` `E_MUT_CONFLICT` |
| 3 | Module resolution, qualification, orphan rule, Ast→ExprInner | `mr-resolve-program-full` / `module_resolver.zyl`, `qualify.zyl`, `expr_inner.zyl` (`convert-ast`), package modules | `E_MODULE_*`, `E_PKG_*` |
| 4 | Macro expansion | `me-expand-program` / `macro_expand.zyl` (`me-collect`, `me-strip`, `me-rewrite`) | `E_MACRO_*`, arity, duplicate, unbound |
| 5 | Checks | `compile-run-checks`: `cc-` capability, `dc-` duplicate, `ac-` arity, `mc-` mutability, `ec-` exhaustiveness, `uc-` unused (W_ to stderr), `sc-` secret | respective codes |
| 6 | Type inference | `collect-definitions` / `type_system.zyl`, `type_inference.zyl` | `E_INVALID_CAPABILITY` only |
| 7 | Monomorphization | `monomorphize` / `monomorphization.zyl` | |
| 8 | Trait dispatch | `td-expand-program` / `trait_dispatch.zyl` | |
| 9 | Closure inlining | `ci-expand-program` / `closure_inline.zyl` (identity pass) | |
| 10 | Assert lowering | `al-expand-program` / `assert_lowering.zyl` | |
| 11 | ICNF lowering | `ic-program` / `icnf.zyl` (`ic-expr-node`, `ic-lambda`, `ic-hoist`, `ic-ffi`) | `E_MATCH_ARM_COMPLEX`, `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` |
| 12 | Optimization | `opt-optimize-fns` / `optimization.zyl` | |
| 13 | Region inference | `ri-transform-fns` / `region_inference.zyl` | |
| 14 | Codegen | `cg-program` / `codegen.zyl` | `E_UNBOUND_VARIABLE` (located), `E_CODEGEN_BUFFER_FULL` |
| 15 | Link | `cc -no-pie out.s actor_runtime.c -o out -lpthread` (driver/`cli.zyl`) | linker errors |
| — | `zyl build` extras | native objects before link; `<name>.buildinfo` after | `E_PKG_NATIVE_*` |

`compile-to-exprs` = stages 1–5; `compile-to-fns` = through 13 (used by REPL and `zyl eval` → `stdlib/repl/interp.zyl`); `compile-to-asm` adds 14. Not wired: `contract_injection.zyl`.

Differences from the spec: module resolution and checks are extra phases before inference; region inference runs last on ICNF; type inference is definition collection feeding monomorphization; phase 10 missing; phase 11 only for package builds (`.buildinfo`: compiler hash, graph hash, empty native-objects, asm hash instead of ICNF hash). Phase isolation holds.

## Compiler module map (37 modules)

Front end `lexer parser ast expr_inner sexp_balance` · packages `module_resolver qualify package store workspace lock index mvs cli capability_check resolver` · `macro_expand` · checks `duplicate_check arity_check mutability_check exhaustiveness_check unused_check secret_check` · types `type_system type_inference` · middle `monomorphization trait_dispatch closure_inline assert_lowering` · back `icnf optimization region_inference codegen` · support `pipeline error_codes error_report` · unwired `contract_injection`.

## Driver and bundle

`selfhost/driver.zyl` (CLI, `drv-usage`) · `selfhost/lsp_main.zyl` (zyl-lsp) · `selfhost/assemble.py` → `selfhost/zyl_selfhost_compiler.zyl` (bundle) · seed `build/boot/stage2.s` / `stage2.bin` · wrapper `build/boot/zyl-self` · runtime `runtime/actor_runtime.c` (actors, arenas, pin, strings, span table, test harness, mangler `zyl_mangle_key`, big stack `zyl_call_on_big_stack`).

## Key data types

`Token` (`Tk*`, last field = byte offset) · `Ast` (`AstIdent AstInt AstFloat AString AstBool AList`) · `Expr`/`ExprInner` (83 variants: `EAtom ECall EApply EDefn ELet ELetMut EIf EMatch EFn EStructGet EDeftype ESpawn ESend ETryCatch ELoadByte EAtomicCAS …`), `Atom`, `Param (P name (Option type))`, `MatchArm (MA variant pats body)` · `Type` (`TInt TFloat TBool TString TUnit TByte TByteSlice TByteBuf TFun TList TArray TCap TStruct TVar TMap TResult`), `CapKind`, `Subst`, `TypeEnv`, `TypeInferer` · `MonoCtx` · `Icnf`/`IArm` · `CGState (CGS …)`, `CGR`, `CGE`, `CGP`.
