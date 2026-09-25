# Compilation Pipeline

## As specified (§22, strict order, no back edges)

1 Parsing → 2 Macro expansion → 3 Type inference + trait resolution → 4 Region inference + capture analysis → 5 Monomorphization → 6 ICNF → 7 Optimization (safe only) → 8 Codegen → 9 Linking → 10 Contract injection (optional) → 11 Hash finalization. §31.9 puts capability enforcement after module resolution, before type inference.

## As implemented (`stdlib/compiler/pipeline.zyl`)

| # | Stage | Function / module | Can raise |
|---|---|---|---|
| 1 | Balance check | `compile-check-balance` / `sexp_balance.zyl` (`sb-check-string`, `sb-hint`) | `E_UNBALANCED_*`, `E_UNTERMINATED_STRING` |
| 2 | Lex + parse (no-dispatch: every form a generic list; `'` `` ` `` `,` `,@` become `quote`/`quasiquote`/`unquote`/`unquote-splicing` lists, `[...]` a `list`) | `zyl-lex`, `zyl-parse-file`, `check-lexed-to-end` / `lexer.zyl`, `parser.zyl`, `ast.zyl` | `E_INVALID_CHAR`, `E_UNTERMINATED_STRING`, `E_INVALID_ESCAPE`, `E_BYTE_VALUE_OOB` |
| 3 | Module resolution, qualification (list literals and well-formed quasiquotes rewritten to `Cons`/`Nil`/`zyl-qq-append`), `impl-not`, orphan rule, Ast→ExprInner | `mr-resolve-program-full` / `module_resolver.zyl`, `qualify.zyl`, `expr_inner.zyl` (`convert-program`, `convert-ast`), package modules | `E_MODULE_*`, `E_PKG_*`, `E_MALFORMED_PARAMETER`, `E_MALFORMED_FORM` (quoted names, bad quasiquote), `E_NESTED_PATTERN`, `E_MATCH_NONEXHAUSTIVE` (literal match), `E_REGION_SPEC`, `E_UNEXPECTED_TOKEN_IN_EXPR`, field `set!` `E_MUT_CONFLICT` |
| 4 | Macro expansion (`&rest`, `,@` splicing) | `me-expand-program` / `macro_expand.zyl` (`me-collect`, `me-strip`, `me-rewrite`) | `E_MACRO_*`, arity, duplicate, unbound, `E_MALFORMED_FORM` |
| 5 | Checks | `compile-run-checks`: `cc-` capability, `dc-` duplicate, `ac-` arity (also `E_MALFORMED_FORM` for any leftover malformed form or stray `,`/`,@`, `ffi-check-call`, `E_FFI_RESTRICTED`), `mc-` mutability, `ec-` exhaustiveness, `uc-` unused (W_ to stderr), `sc-` secret | respective codes |
| 6 | Derive expansion | `dv-expand-program` / `derive.zyl` (Show/Debug/Eq/Ord/Hash/Clone → impls; `dv-check-fields`) | `E_TRAIT_NOT_DERIVABLE`, `E_DUPLICATE_IMPL`, `E_IMPL_FORBIDDEN` |
| 7 | Impl lifting | `lift-impls` / `lift_impls.zyl` (each method becomes `Trait.method_Type`) | |
| 8 | Closure inlining | `ci-expand-program` / `closure_inline.zyl` (retired; identity pass) | |
| 9 | Type checking: sound HM, static trait resolution, per-type specialization of calls and function values, generated structural `T.==`; every error reported, then the compile fails | `ta-annotate` / `type_annotate.zyl`, runtime signatures `ffi_sigs.zyl`; results in `node_tables.zyl` (`node-types`, `node-calls`, `node-shows`) | `E_TYPE_MISMATCH`, `E_INFINITE_TYPE`, `E_CANNOT_INFER`, `E_UNBOUND_VARIABLE`, `E_TRAIT_NOT_FOUND`, `E_FFI_TYPE_NOT_PINNABLE`, `E_FFI_RESTRICTED` (extern), `E_MALFORMED_PARAMETER` (trait as type) |
| 10 | ICNF lowering | `ic-program` / `icnf.zyl` (`ic-expr-node`, `ic-lambda`, `ic-hoist`, `ic-ffi`, `ic-with-region`; sets `icnf-kinds`, `icnf-scalars`, `icnf-adts`) | `E_MATCH_ARM_COMPLEX`, `E_MATCH_NONEXHAUSTIVE`, `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` |
| 11 | Inlining + copy propagation | `opt-inline-fns` / `optimization.zyl` (two rounds; `ZYL_INLINE=0`, `ZYL_INLINE_LIMIT`) | |
| 12 | Optimization | `opt-optimize-fns` / `optimization.zyl` (int constant folding, dead-branch elimination) | |
| 13 | Region inference | `rg-regions (ri-transform-fns ...)` / `region_inference.zyl`: the `IStackVariant` rewrite, then whole-program escape analysis classifying every allocation and call site as frame (L), result (R) or heap (H) into `icnf-regions` (reads `icnf-scalars`); `ZYL_REGIONS=0` makes every site H | `E_REGION_ESCAPE` |
| 14 | Reuse | `ru-reuse` / `reuse.zyl`: marks a construction that may take a unique, dead value's block (`icnf-reuses`), adds owning clones `f~own`; `ZYL_REUSE=0` off, `ZYL_REUSE_DEBUG` dumps facts | |
| 15 | Codegen | `cg-program-file` / `codegen.zyl`: per function, `mb-eligible` picks the native backend (lower to MIR, `mir.zyl` liveness + linear scan, emit) or the stack machine; `ZYL_MIR=0` forces the stack machine | `E_UNBOUND_VARIABLE` (located fallback), `E_CODEGEN_BUFFER_FULL` |
| 16 | Link | `cc -no-pie out.s actor_runtime.c -o out -lpthread` (`selfhost/driver.zyl`) | linker errors |
| — | `zyl build`/`zyl test` extras | lowers first (`compile-to-fns`) to hash the canonical ICNF text (`icnf-text`, `icnf_print.zyl`); native objects before link; final hash appended to the assembly as `zyl_build_hash` (section `.zyl_build`); `<name>.buildinfo` after (`drv-compile-file`, `drv-build-hashes`) | `E_PKG_NATIVE_*` |

`compile-to-exprs` = stages 1–5; `compile-to-fns` = through 14 (used by the REPL and `zyl eval`, then `stdlib/repl/interp.zyl`, which ignores region and reuse annotations); `compile-to-asm` adds 15. Contracts are lowered to checks in `expr_inner.zyl` under the active profile (`--contracts=P`, `(contracts P)`); there is no contract-injection stage. `ZYL_DEBUG_STAGES=1` appends each stage name to `/tmp/dbg`.

Differences from the spec: module resolution and the checks are extra phases before type checking; type checking runs after derive expansion and impl lifting, on the final Expr program, and replaces monomorphization (specialization happens inside it); inlining, region inference and reuse run on ICNF; phase 10 folded into the front end; phase 11 only for package builds. Phase isolation holds.

`<name>.buildinfo` (§31.12, no remaining deviation):

```lisp
(buildinfo
  (compiler-hash "blake3:...")   ; the compiler binary
  (graph-hash "blake3:...")      ; from zyl.lock; empty without a lock
  (graph (package "acme/json" "1.4.0" "blake3:..."))   ; resolved graph from the lock (lk-graph-text), sorted
  (native-objects ("build/native/c_fast.c.o" "blake3:..."))  ; package-relative, manifest order
  (icnf-hash "blake3:...")       ; canonical ICNF text, codegen kinds included
  (asm-hash "blake3:...")        ; emitted assembly (informational, not in the final hash)
  (final-hash "blake3:..."))     ; over compiler, graph, native and ICNF hashes; = zyl_build_hash in the binary
```

`objdump -s -j .zyl_build app` shows the embedded hash; the same package built in two directories gives byte-identical binaries. A plain `zyl file.zyl` writes no buildinfo.

## Compiler module map (41 modules)

Front end `lexer parser ast expr_inner sexp_balance` · packages `module_resolver qualify package store workspace lock index mvs cli capability_check resolver` (the last mostly dead code) · `macro_expand` · checks `duplicate_check arity_check mutability_check exhaustiveness_check unused_check secret_check` · types `type_annotate` (the checker), `ffi_sigs` (runtime signatures, `ffi-raw-p`), `type_system` (only `Pair` and `Region`) · middle `derive lift_impls closure_inline` · back `icnf icnf_print optimization region_inference reuse mir codegen` · support `pipeline error_codes error_report node_tables doc` (`zyl doc`). `type_inference`, `monomorphization` and `assert_lowering` were deleted on 2026-09-25.

## Driver and build

`selfhost/driver.zyl` (CLI, `drv-usage`: `zyl <file.zyl> [-o] [--emit-asm] [--error-format=json] [--contracts=P]`, `new add fetch build test update vendor audit publish key repl eval doc`; package builds go through the content-hash cache in `drv-compile-file`) · `selfhost/lsp_main.zyl` (zyl-lsp) · the compiler is built from `selfhost/driver.zyl` through module resolution (no bundle since 2026-09-24); `boot.sh` caps each stage at 4 GB (`ZYL_STAGE_MEMORY`) · seed `build/boot/stage2.s` / `stage2.bin` · wrapper `build/boot/zyl-self` · runtime `runtime/actor_runtime.c` (actors, arenas, regions, pin, strings, typed arrays, span table, test harness, mangler `zyl_mangle_key`, big stack `zyl_call_on_big_stack`, timed FFI `zyl_ffi_timed`, `zyl_div_magic`).

## Key data types

`Token` (`Tk*`, last field = byte offset; includes `TkQuote TkQuasi TkUnquote TkSplice`) · `Ast` (`AstIdent AstInt AstFloat AString AstBool AList`) · `Expr`/`ExprInner` (83 variants: `EAtom ECall EApply EDef EDefn ELet ELetMut EIf EMatch EFn EStructGet EDeftype ESpawn ESend ETryCatch ELoadByte EAtomicCAS … EUnknown`), `Atom`, `Param (P name (Option type))`, `MatchArm (MA variant pats body)`, `Endian (ELe) (EBe) (EWide code width)` · types `TaTy (TaV Int) (TaC String (List TaTy)) (TaF (List TaTy) TaTy)` and the checker state `TaSt` (`type_annotate.zyl`) · `Icnf` (21 variants: `IConst IStr IFlt ILoad IBinop ICall IFfi IPrint IIf IWhile ISet ILet ISeq IVariant IMatch IFn ICallClosure ITryCatch IStackVariant ISymAddr IRegion`) / `IArm` · `MI` (MIR instructions: `MConst MMov MBin MBinI MParam MCall MTail MByte … MReuse MByteFast`), `Iv` (live interval), `Alloc` (`mir.zyl`) · `CGState (CGS …)`, `CGR`, `CGE`, `CGP`, native-path `MS MR MCtx MF` (`codegen.zyl`).
