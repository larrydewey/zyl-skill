# Error Codes

Catalog of record: `stdlib/compiler/error_codes.zyl` (name, phase, severity, message). Formatting: `stdlib/compiler/error_report.zyl`. Every check stops at its first error; one diagnostic per compile.

Two shapes, plus labelled secondary spans on some located errors, and JSON with `--error-format=json`:

```text
error[E_MALFORMED_PARAMETER]: `(struct-get ...)` is not a parameter
  --> tests/regression/structs.zyl:16:20
   |
16 | (defn _s-get-x (p (struct-get p "x"))
   |                   ^
   = help: a missing `)` earlier on the line puts the body in the list

PANIC: E_MATCH_ARM_COMPLEX: ...           (unlocated)
```

```text
error[E_MUT_CONFLICT]: set! target `x` is not a let-mut binding in scope
  --> prog.zyl:3:5
   |
 3 |     (set! x 2)
   |     ^
 1 | (let x 1
   | - bound here by `let`, which is immutable (TCap)
   = help: only a let-mut binding is TMut and may be assigned; declare it with `let-mut`
```

`E_UNBOUND_VARIABLE` adds `= help: did you mean `name`?` when a visible name is within edit distance.

Legend: **R** raised · **C** catalogued only (never raised) · **U** raised but not catalogued · **I** interpreter/REPL only · **W** warning.

## Lexing and parsing

| Code | Status | Trigger → fix |
|---|---|---|
| `E_UNTERMINATED_STRING` | R | missing closing `"` |
| `E_BYTE_VALUE_OOB` | R | `(byte n)` outside 0..255 or non-integer |
| `E_UNBALANCED_UNCLOSED` | R | opener never closed (line/col + fix-it) |
| `E_UNBALANCED_UNEXPECTED_CLOSE` | R | stray closer |
| `E_UNBALANCED_MISMATCHED_BRACKET` | R | `(` closed by `]` etc. |
| `E_UNBALANCED_OPEN_STRING` | U | LSP balance check: unterminated string |
| `E_MALFORMED_PARAMETER` | R | param not a name or `(name Type)`; `((T) x)`; body swallowed by param list; non-identifier macro param/name-position arg |
| `E_UNEXPECTED_TOKEN_IN_EXPR` | R | token illegal in expression position |
| `E_RESERVED_KEYWORD` | R | only for `load-u16/u32/u64`, signed and store variants |
| `E_INVALID_CHAR`, `E_UNEXPECTED_EOF`, `E_INTEGER_OVERFLOW`, `E_FLOAT_OVERFLOW`, `E_UNBALANCED_PARENS`, `E_EXPECTED_RPAREN/RBRACKET/RCURLY`, `E_EXPECTED_EXPRESSION`, `E_EMPTY_LIST`, `E_ATOM_AS_OPERATOR` | C | stray chars **truncate silently**; oversized ints become **0** |

## Macros

| Code | Status | Trigger |
|---|---|---|
| `E_MACRO_NON_TERMINATION` | R | macro reached during its own expansion; > 256 nesting |
| `E_MACRO_ILLEGAL_ACCESS` | R | `defmacro` not at top level |
| also `E_ARITY_MISMATCH`, `E_DUPLICATE_DEFINITION`, `E_MALFORMED_PARAMETER`, `E_UNBOUND_VARIABLE` (template names a call-site local) | R | |

## Names, arity, types

| Code | Status | Trigger |
|---|---|---|
| `E_UNBOUND_VARIABLE` | R | unbound name; top-level `def` reference; bare `:keyword`; return-type slot; `((x) ...)` pseudo-lambda |
| `E_ARITY_MISMATCH` | R | wrong arg count; `bit-not` ≠ 1 arg; `(T : Ord)` params; guard after `range` |
| `E_DUPLICATE_DEFINITION` | R | top-level name twice; redefining prelude names/types; macro+fn same file |
| `E_DUPLICATE_VARIANT` | R | variant repeated in one `deftype` |
| `E_DUPLICATE_PARAMETER` | U | repeated param (`_`/`_x` exempt), from `unused_check` |
| `E_INVALID_CAPABILITY` | R | non-FFI_Pinnable `ffi-pin` operand / inline closure in `ffi-call` |
| `E_TYPE_MISMATCH` | R | argument definitely clashing with a top-level function's parameter annotation or a constructor's field type (`type_annotate`); nothing else |
| `E_RETURN_TYPE_MISMATCH`, `E_UNKNOWN_TYPE`, `E_UNKNOWN_GENERIC_PARAM`, `E_CANNOT_INFER` | C | other type errors are **not rejected** |

## Matching

| Code | Status | Trigger |
|---|---|---|
| `E_NON_EXHAUSTIVE_MATCH` | U | constructor match missing a variant, no `_` |
| `E_UNREACHABLE_MATCH_ARM` | U | arm after a catch-all (incl. misspelled/unknown constructor) |
| `E_MATCH_NONEXHAUSTIVE` | R | literal match without trailing `_` (spec spelling) |
| `E_MATCH_ARM_COMPLEX` | R | arm body: constant + ≥ 2 calls in one binop |

## Capabilities, aliasing, secrets

| Code | Status | Trigger |
|---|---|---|
| `E_MUT_CONFLICT` | R | `set!` on non-`let-mut`, parameter, struct field, or captured var inside a closure |
| `E_CAPABILITY_LEAK` | R | `let-mut` name in `send` message or `spawn` body |
| `E_CT_VIOLATION` | R | Secret in branch/match subject, index/offset, `/`/`%` |
| `E_SECRET_DEBUG` | R | Secret to `print` |
| `E_SECRET_ESCAPE` | R | Secret to `spawn`/`send`/`file-write` |
| `E_FFI_PIN_REQUIRED` | R | Secret raw `ffi-call` arg |
| `E_ZEROIZE_MISSING` | W | Secret param consumed into public result without zeroize |
| `E_PKG_CAPABILITY_VIOLATION` | R | undeclared capability in a package (not `main`/tests) |
| `E_PKG_CAPABILITY_GROWTH` | R | closure grew under `--locked` |

## Regions, bytes, lowering, codegen, runtime

| Code | Status | Notes |
|---|---|---|
| `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` | R | tests/top-level statements + `defn main` (incl. from a `use`d module) |
| `E_CODEGEN_BUFFER_FULL` | R | > 63 MiB assembly |
| `E_OUT_OF_MEMORY` | R | allocation failed / budget (`ZYL_MAX_MEMORY`, 0 disables; default 80% RAM) |
| `E_LIST_NTH_OOB` | R | compiler-internal `list-nth` |
| `E_DIVISION_BY_ZERO` | I | compiled code gets SIGFPE instead |
| `E_REGION_ESCAPE`, `E_UNINITIALIZED_USE`, `E_ATOMIC_ABA`, `E_BYTEBUF_NOT_PIN`, `E_STACK_BYTEBUF_RETURN`, `E_GLOBAL_BYTEBUF_MUT` | C | |
| `E_CODEGEN`, `E_CODEGEN_BUFFER_LIMIT` | C | |
| `E_USER_ERROR` | C | `error` prints `PANIC: <msg>` instead |
| `E_ASSERT_FAIL` | C | a failing `assert` panics with `assert failed`, no code |
| `E_NULL_POINTER`, `E_BYTE_OOB`, `E_BYTEBUF_CAP_EXCEEDED`, `E_BYTEBUF_OVERLAP`, `E_BYTEBUF_INVALID`, `E_ALIGNMENT_FAILED`, `E_ALIGN_CHECK_FAILED` | C | byte ops fail closed returning 0 |
| `E_OVERFLOW` | C | ints wrap |
| `E_CONTRACT_VIOLATION` | C | contracts not enforced |
| `E_FFI_TIMEOUT`, `E_FFI_TYPE_NOT_PINNABLE` | C | timeouts dropped; pinnability → `E_INVALID_CAPABILITY` |
| `E_TEST_FAILURE`, `E_TEST_RUNNER_ERROR` | C | tests print `FAIL` |
| `E_UNDEFINED_FUNCTION`, `E_NOT_CALLABLE`, `E_UNSUPPORTED_INTERPRETED`, `E_FFI_SYMBOL_NOT_FOUND`, `E_NO_MAIN`, `E_INTERNAL` | I/U | REPL interpreter / evaluator |

## Traits

| Code | Status | Actual behavior |
|---|---|---|
| `E_PKG_ORPHAN_IMPL` | R | impl where neither trait nor type is local |
| `E_TRAIT_NOT_FOUND` | C | link-time undefined reference |
| `E_DUPLICATE_IMPL` | C | assembler "symbol already defined" |
| `E_TRAIT_BOUND_NOT_SATISFIED`, `E_TRAIT_NOT_DERIVABLE` | C | bounds unwritable; derive does not check fields |

## Modules and packages (all R)

`E_MODULE_NOT_FOUND` (lone file), `E_MODULE_CYCLE`, `E_PKG_CYCLE`, `E_PKG_UNKNOWN_MODULE`, `E_PKG_UNDECLARED_DEP` (also a missing module of the current package), `E_PKG_PRIVATE_SYMBOL`, `E_PKG_UNKNOWN_SYMBOL`, `E_PKG_RESERVED_MODULE`, `E_MANIFEST_INVALID`, `E_MANIFEST_NOT_FOUND`, `E_PKG_BAD_NAME`, `E_PKG_BAD_VERSION`, `E_PKG_BAD_REQUIREMENT` (range operator), `E_PKG_DUPLICATE_DEP`, `E_PKG_VERSION_CONFLICT`, `E_PKG_NOT_FOUND`, `E_PKG_VERSION_NOT_FOUND`, `E_PKG_NOT_IN_STORE`, `E_PKG_HASH_MISMATCH`, `E_PKG_SIGNATURE_INVALID`, `E_PKG_KEY_CHANGED`, `E_PKG_UNSIGNED`, `E_PKG_YANKED`, `E_PKG_LOCK_STALE`, `E_PKG_LOCK_INVALID`, `E_PKG_COMPILER_TOO_OLD`, `E_PKG_UNKNOWN_EDITION`, `E_PKG_FETCH_FAILED`, `E_PKG_ARCHIVE_INVALID`, `E_PKG_NATIVE_PATH_ESCAPE`, `E_PKG_NATIVE_FLAG_DENIED`, `E_PKG_NATIVE_BUILD_FAILED`, `E_PKG_FEATURE_UNKNOWN`, `E_PKG_FEATURE_COLLISION`. Catalogued synonyms never raised: `E_CIRCULAR_MODULE`, `E_SYMBOL_NOT_EXPORTED`. Format: `PANIC: CODE: area: message`.

## Warnings (stderr, never fatal, not shown by the LSP)

`W_UNUSED_PARAMETER`, `W_UNUSED_VARIABLE`, `W_SHADOWED_BINDING`, `E_ZEROIZE_MISSING`. `W_UNUSED_FUNCTION` is implemented but not wired in (it would flag every unused auto-injected `core` function). `_`/`_`-prefixed names are exempt. Printed as located `warning[CODE]` diagnostics through the runtime's warning sink (`zyl_warn_emit`), which a caller can capture instead of printing.

## Exit codes

0 success · 1 compile error or runtime panic · 139 SIGSEGV (compiler/runtime bug) · 134 SIGABRT (internal error, e.g. `send` to a stopped actor) · SIGFPE integer division by zero.
