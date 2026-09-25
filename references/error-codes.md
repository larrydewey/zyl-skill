# Error Codes

Catalog of record: `stdlib/compiler/error_codes.zyl` (name, phase, severity, message). Formatting: `stdlib/compiler/error_report.zyl`. Spec §28 lists the required codes. Every check except the type pass stops at its first error. The type pass (`type_annotate.zyl`) reports **every** type error it finds (`E_TYPE_MISMATCH`, `E_INFINITE_TYPE`, `E_CANNOT_INFER`, `E_UNBOUND_VARIABLE`, `E_TRAIT_NOT_FOUND`, `E_FFI_TYPE_NOT_PINNABLE`, `E_FFI_RESTRICTED` for an `extern` of a runtime entry, `E_MALFORMED_PARAMETER` for a trait used as a type), then fails with `PANIC: error[CODE]: the program does not type-check (N errors above)`, CODE being the first error's. `ZYL_STRICT_TYPES=report` turns them into `W_TYPE_STRICT` warnings and continues: a counting tool, never a way to ship ill-typed code.

Two shapes, plus labelled secondary spans on some located errors, and JSON with `--error-format=json`:

```text
error[E_MALFORMED_PARAMETER]: `(struct-get ...)` is not a parameter
  --> tests/regression/structs.zyl:16:20
   |
16 | (defn _s-get-x (p (struct-get p "x"))
   |                   ^
   = help: a missing `)` earlier on the line puts the body in the list

PANIC: E_INVALID_CAPABILITY: ...          (unlocated)
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
| `E_INVALID_CHAR` | R | a byte that cannot start a token outside strings/comments, located: `@` (except in `,@`), `#`, `$`, `\|`, `^`, `\`, a `.` not followed by a letter, a control character, any non-ASCII byte, a BOM. `'` `` ` `` `,` `,@` are the quote/quasiquote/unquote/splice tokens, and `&` may start an identifier (`&rest`) |
| `E_UNTERMINATED_STRING` | R | missing closing `"`; the balance check reports it first, unlocated ("reached end of input") |
| `E_INVALID_ESCAPE` | R | an escape other than `\n \t \r \0 \" \\ \e \xNN` in a string literal, located |
| `E_BYTE_VALUE_OOB` | R | `(byte n)` outside 0..255 or non-integer; unlocated |
| `E_UNBALANCED_UNCLOSED` | R | opener never closed (line/col + fix-it) |
| `E_UNBALANCED_UNEXPECTED_CLOSE` | R | stray closer |
| `E_UNBALANCED_MISMATCHED_BRACKET` | R | `(` closed by `]` etc. |
| `E_UNBALANCED_OPEN_STRING` | U | LSP balance check: unterminated string |
| `E_MALFORMED_PARAMETER` | R | param not a name or `(name Type)`; `(a : Int)` ("write (a Int)"); `((T : Ord) x)`; a trait in type position, `(a Ord)` (`Secret` exempt; reported by the type pass); body swallowed by the param list; non-identifier macro param, misplaced `&rest`, non-identifier argument in a macro name position |
| `E_MALFORMED_FORM` | R | a special form whose arguments have the wrong shape (it used to lower to the constant 0): `(let ((x 1) (y 2)) ...)`, `(let x 1)` with no body, `(if c)`, `(make-variant 1 2)`, a `defmacro` or `test` with more than one body form, a bare-`self` trait signature `(area self)`, `(quote a b)`; a name inside quoted data, `'(1 a)`, or outside an unquote in a quasiquote; a nested quasiquote; a `,` or `,@` outside a quasiquote and a macro template; in a template, `,@` where a fixed number of expressions is taken (an `if`, constructor fields, which include `(list ,@xs)` and `[,@xs]`) or of anything but the `&rest` parameter; a form after an `if`'s else branch; a quote or unquote with nothing after it, and a non-name in an import list (`{ a, b }`). Raised by `arity_check`, `expr_inner`, `parser`, `module_resolver` and `macro_expand`; located |
| `E_UNEXPECTED_TOKEN_IN_EXPR` | R | load/store endian not `:le`/`:be`; bytebuf capacity not an integer literal or region not a region name; unlocated |
| `E_RESERVED_KEYWORD` | C | catalogued (spec §28), not raised |
| `E_UNEXPECTED_EOF`, `E_INTEGER_OVERFLOW`, `E_FLOAT_OVERFLOW`, `E_UNBALANCED_PARENS`, `E_EXPECTED_RPAREN/RBRACKET/RCURLY`, `E_EXPECTED_EXPRESSION`, `E_EMPTY_LIST`, `E_ATOM_AS_OPERATOR` | C | an integer literal ≥ 2^63 silently becomes **0** |

## Macros

| Code | Status | Trigger |
|---|---|---|
| `E_MACRO_NON_TERMINATION` | R | macro reached during its own expansion; > 256 nesting |
| `E_MACRO_ILLEGAL_ACCESS` | R | `defmacro` not at top level |
| also `E_ARITY_MISMATCH` (wrong count, too few before `&rest`), `E_DUPLICATE_DEFINITION`, `E_MALFORMED_PARAMETER`, `E_MALFORMED_FORM` (misplaced `,@`, several body forms), `E_UNBOUND_VARIABLE` (template names a call-site local) | R | |

## Names, arity, types

| Code | Status | Trigger |
|---|---|---|
| `E_TYPE_MISMATCH` | R | any unification failure, both types in the message: a non-`Bool` condition (`if`, `cond`, `while`, `and`/`or`/`not`, guards, `assert`), `Int` and `Float` mixed in arithmetic, `if` without `else` whose `then` is not `Unit`, arms or branches of two types, `main` not `() -> Int`, list-literal elements of two types, an argument against a parameter or field annotation, `<` on a struct/ADT ("ordering on P"), a `Result` given to `unwrap`, a non-literal `file-open` mode, non-`Int` byte offsets/values, a slice where a `ByteBuf` is needed, a spawn entry with parameters, `Float` or a type variable in an `extern` |
| `E_INFINITE_TYPE` | R | occurs check: `(x x)` |
| `E_CANNOT_INFER` | R | `ffi-call` to a foreign symbol with no `(extern ...)`; a runtime entry with no signature in `ffi_sigs.zyl` (`zyl_actor_send_closure`: "no type for untyped ffi result"); a trait call whose receiver type never resolves; a byte load/store handle that may be `ByteBuf` or `ByteSlice`; a function needing more than 256 specialized instances; `make-variant`/`make-struct` |
| `E_UNBOUND_VARIABLE` | R | unbound name or undefined function (the type pass reports every one; codegen keeps a located fallback); bare `:keyword`; `((x) ...)` pseudo-lambda; a `let` binding used after its `let` form ends; a call to a `defun` (not recognized, so its name is undefined) or to `tuple` |
| `E_ARITY_MISMATCH` | R | wrong arg count; `/`, `%`, `bit-and/or/xor` with one operand (`operator 3 needs two operands`); `bit-not` ≠ 1 arg; a guard after `range` or on a `_` arm (read as the 2-arg `when`); malformed byte/load/store/atomic forms; `ffi-call` with more than 16 args |
| `E_DUPLICATE_DEFINITION` | R | top-level name twice; redefining prelude names/types; macro+fn same file |
| `E_DUPLICATE_VARIANT` | R | variant repeated in one `deftype`; a program type reusing a prelude constructor (`Some None Ok Err Cons Nil`) |
| `E_DUPLICATE_PARAMETER` | U | repeated param (`_`/`_x` exempt), from `unused_check` |
| `E_INVALID_CAPABILITY` | R | a closure written inline as an `ffi-call` argument (`mutability_check`); unlocated |
| `E_UNKNOWN_TYPE` | R | a lowercase field type in `deftype`, `(Bx a)` (neither a type nor an uppercase parameter); located |
| `E_RETURN_TYPE_MISMATCH`, `E_UNKNOWN_GENERIC_PARAM` | C | an unknown capitalized type name in an annotation is **not** an error: it becomes a type parameter (`(a Intt)` accepts any type) |

## Matching

| Code | Status | Trigger |
|---|---|---|
| `E_NON_EXHAUSTIVE_MATCH` | U | constructor match missing a variant, no `_` (`exhaustiveness_check`) |
| `E_UNREACHABLE_MATCH_ARM` | U | arm after a catch-all (incl. misspelled/unknown constructor), or a repeated constructor arm |
| `E_MATCH_NONEXHAUSTIVE` | R | literal match without trailing `_`; from lowering, a variant missed when the subject's ADT is only known there (shared variant names); unlocated |
| `E_NESTED_PATTERN` | R | a constructor field that is itself a pattern, `(Some (Cons x _) ...)`, including a guard on a constructor arm, `(Some x (when c) ...)`, and a known constructor in a binder slot, `(Node v Leaf v)`; located |
| `E_MATCH_ARM_COMPLEX` | R | arm body: one binop with a constant and ≥ 2 calls, `(+ 1 (g x) (g x))`; located |

## Capabilities, aliasing, secrets

| Code | Status | Trigger |
|---|---|---|
| `E_MUT_CONFLICT` | R | `set!` on non-`let-mut`, parameter, struct field, or captured var inside a closure |
| `E_CAPABILITY_LEAK` | R | `let-mut` name in `send` message or `spawn` body |
| `E_CT_VIOLATION` | R | Secret in branch / subject of a multi-arm match (one-arm destructuring is allowed), index/offset, `/`/`%` |
| `E_SECRET_DEBUG` | R | Secret to `print`, into an `error` message, or into the text a `show` impl returns (impl bodies are checked); a secret inside a `Secret` field of a printed record is redacted, not an error |
| `E_SECRET_ESCAPE` | R | Secret to `spawn`/`send`/`file-write` |
| `E_ZEROIZE_MISSING` | W | Secret param consumed into public result without zeroize |
| `E_PKG_CAPABILITY_VIOLATION` | R | undeclared capability in a package (not `main`/tests) |
| `E_PKG_CAPABILITY_GROWTH` | R | closure grew under `--locked` |

Secret-checker errors are located `error[CODE]: in `f`: ...` diagnostics (file:line:col); a `let-mut` ever `set!` to a secret is secret for its whole scope.

## FFI

| Code | Status | Trigger |
|---|---|---|
| `E_FFI_SYMBOL_REQUIRED` | R | `ffi-call` symbol not a string literal (`ffi-check-call`, `arity_check.zyl`) |
| `E_FFI_TIMEOUT_REQUIRED` | R | `ffi-call` last arg not a positive integer literal (missing, variable, 0, negative); located, with help |
| `E_FFI_RESTRICTED` | R | a raw runtime entry (`ffi-raw-p` in `ffi_sigs.zyl`: `zyl_word_load`, `zyl_mem_read`, `zyl_cstr_of_word`, `zyl_view_*`, `zyl_val_*`, ...) named in an `ffi-call` outside the standard library (arity check, located); an `extern` that retypes a `zyl_*` runtime entry (type pass, at each call) |
| `E_FFI_PIN_REQUIRED` | R | Secret raw `ffi-call` arg |
| `E_FFI_TYPE_NOT_PINNABLE` | R | `ffi-pin` of a function, located at the operand (type pass) |
| `E_FFI_TIMEOUT` | R | runtime panic from `zyl_ffi_timed`: ``E_FFI_TIMEOUT: ffi call `sym` exceeded its timeout of N ms``; catchable with `try`, matched by `recover`/`zyl_err_is`; the C call is abandoned |
| `E_FFI_SYMBOL_NOT_FOUND` | U | runtime symbol lookup for the interpreter (`zyl eval`, REPL) |

## Regions, bytes, lowering, codegen, runtime

| Code | Status | Notes |
|---|---|---|
| `E_REGION_ESCAPE` | R | located, from region inference (`rg-*`): a `(bytebuf Stack N)` returned, stored, sent or passed to code that may keep it; a value allocated inside `with-region` that outlives it. An inferred placement never raises it |
| `E_REGION_SPEC` | R | located, from `parse-with-region`: unknown kind (only `arena`, `fixed`), `:block` not a multiple of 4096 or above 64 MiB, `:align` not a power of two in 8..4096 |
| `E_REGION_EXHAUSTED` | R | runtime panic when a `fixed` region's `:size` or an arena's `:limit` is exceeded; deterministic, catchable with `try`, matched by `recover`/`zyl_err_is`; compiled code only (the interpreter does not enforce limits) |
| `E_INDEX_OUT_OF_BOUNDS` | R | runtime panic: `vec-get`/`vec-last` outside the Vec, `slice-vec`/`slice-sub`/`slice-get` outside the range (`collections/vec`, `collections/slice`), the runtime's array/vector accessors; catchable with `try` |
| `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` | R | tests/top-level statements + `defn main` (incl. from a `use`d module) |
| `E_CODEGEN_BUFFER_FULL` | R | > 63 MiB assembly |
| `E_OUT_OF_MEMORY` | R | allocation failed / budget (`ZYL_MAX_MEMORY`, 0 disables; default 80% RAM); catalogued twice |
| `E_DIVISION_BY_ZERO` | I | compiled code dies with SIGFPE (exit 136) instead, which `try` cannot catch |
| `E_INTERP_TAG` | R | the checking interpreter (`ZYL_INTERP_CHECK=1`) found a wrong-tag operand or a condition not 0/1: a type-checker bug |
| `E_UNINITIALIZED_USE`, `E_ATOMIC_ABA`, `E_BYTEBUF_NOT_PIN`, `E_STACK_BYTEBUF_RETURN`, `E_GLOBAL_BYTEBUF_MUT` | C | a Stack bytebuf escape reports `E_REGION_ESCAPE` |
| `E_CODEGEN`, `E_CODEGEN_BUFFER_LIMIT`, `E_LIST_NTH_OOB` | C | |
| `E_USER_ERROR` | C | `error` prints `PANIC: <msg>` instead |
| `E_ASSERT_FAIL` | C | a failing `assert` panics with its string-literal message, or `assert failed` |
| `E_NULL_POINTER`, `E_BYTE_OOB`, `E_BYTEBUF_CAP_EXCEEDED`, `E_BYTEBUF_OVERLAP`, `E_BYTEBUF_INVALID`, `E_ALIGNMENT_FAILED`, `E_ALIGN_CHECK_FAILED` | C | byte ops fail closed returning 0; `align-check` returns a Bool |
| `E_OVERFLOW` | C | ints wrap |
| `E_CONTRACT_VIOLATION` | R | failed `requires`/`ensures`/`invariant`: `precondition of f failed: C` (also `postcondition`, `invariant`); under the `warn` profile a stderr `warning: E_CONTRACT_VIOLATION: ...` and execution continues; `off`/`production` compile the checks out; `recover` arms match it by prefix |
| `E_TEST_FAILURE`, `E_TEST_RUNNER_ERROR` | C | tests print `FAIL` |
| `E_UNDEFINED_FUNCTION`, `E_NOT_CALLABLE`, `E_UNSUPPORTED_INTERPRETED`, `E_NO_MAIN`, `E_INTERNAL` | I/U | REPL interpreter / evaluator |

## Traits

| Code | Status | Actual behavior |
|---|---|---|
| `E_PKG_ORPHAN_IMPL` | R | impl where neither trait nor type is local |
| `E_TRAIT_NOT_FOUND` | R | located, from the type pass: a trait call (qualified `(Show.show x)` or dot) on a receiver of known type with no impl (`= help: add (impl Show P ...)`); a dot call whose method no trait declares. There is no run-time dispatch: an unresolved receiver is `E_CANNOT_INFER` |
| `E_IMPL_FORBIDDEN` | R | impl or derive of a pair forbidden by `(impl-not Trait Target)` (located), or an impl of `Trait` whose result derives from a protected value (flow rule, also located, at the exposing expression); prelude `(impl-not Show/Debug/Eq/Ord/Hash Secret)` makes such an impl for a Secret type this error |
| `E_DUPLICATE_IMPL` | R | located: two impls of one trait for one type, or the same trait derived twice (`` `Show` is implemented for `Q` more than once ``) |
| `E_TRAIT_NOT_DERIVABLE` | R | located: `derive` of a trait other than Show/Debug/Eq/Ord/Hash/Clone, a field type that does not implement the trait, or a `Secret` field under `Eq`/`Ord`/`Hash` |
| `E_TRAIT_BOUND_NOT_SATISFIED` | C | bounds are not written; a missing impl is `E_TRAIT_NOT_FOUND` at the use |

## Modules and packages (all R)

`E_MODULE_NOT_FOUND` (lone file), `E_MODULE_CYCLE`, `E_PKG_CYCLE`, `E_PKG_UNKNOWN_MODULE`, `E_PKG_UNDECLARED_DEP` (also a missing module of the current package), `E_PKG_PRIVATE_SYMBOL`, `E_PKG_UNKNOWN_SYMBOL`, `E_PKG_RESERVED_MODULE`, `E_MANIFEST_INVALID`, `E_MANIFEST_NOT_FOUND`, `E_PKG_BAD_NAME`, `E_PKG_BAD_VERSION`, `E_PKG_BAD_REQUIREMENT` (range operator), `E_PKG_DUPLICATE_DEP`, `E_PKG_VERSION_CONFLICT`, `E_PKG_NOT_FOUND`, `E_PKG_VERSION_NOT_FOUND`, `E_PKG_VERSION_EXISTS` (`zyl publish --index`: that version is already in the index; versions are immutable), `E_PKG_NOT_IN_STORE`, `E_PKG_HASH_MISMATCH`, `E_PKG_SIGNATURE_INVALID`, `E_PKG_KEY_CHANGED`, `E_PKG_UNSIGNED`, `E_PKG_YANKED`, `E_PKG_LOCK_STALE`, `E_PKG_LOCK_INVALID`, `E_PKG_COMPILER_TOO_OLD`, `E_PKG_UNKNOWN_EDITION`, `E_PKG_FETCH_FAILED`, `E_PKG_ARCHIVE_INVALID`, `E_PKG_NATIVE_PATH_ESCAPE`, `E_PKG_NATIVE_FLAG_DENIED`, `E_PKG_NATIVE_BUILD_FAILED`, `E_PKG_FEATURE_UNKNOWN`, `E_PKG_FEATURE_COLLISION`, `E_PKG_FEATURE_NESTED` (located: `feature-gate` inside another form; valid at top level only). Catalogued synonyms never raised: `E_CIRCULAR_MODULE`, `E_SYMBOL_NOT_EXPORTED`. Format: `PANIC: CODE: area: message`.

## Warnings (stderr, never fatal; the LSP publishes them as Warning diagnostics)

`W_UNUSED_PARAMETER`, `W_UNUSED_VARIABLE`, `W_SHADOWED_BINDING`, `E_ZEROIZE_MISSING`, and `W_TYPE_STRICT` (only under `ZYL_STRICT_TYPES=report`). `W_UNUSED_FUNCTION` is implemented but not wired in (it would flag every unused auto-injected `core` function). `_`/`_`-prefixed names are exempt. Printed as located `warning[CODE]` diagnostics through the runtime's warning sink (`zyl_warn_emit`), which a caller can capture instead of printing.

## Exit codes

0 success · 1 compile error or runtime panic · 136 SIGFPE (integer `/` or `%` by zero) · 139 SIGSEGV (compiler/runtime bug) · 134 SIGABRT (internal error, e.g. `send` to a stopped actor).
