# Silent-Failure Checklist

Things that compile without complaint and then do the wrong thing. Scan this before shipping or reviewing Zyl code. Each links to the rule with the fix.

Since 2026-09-25 the type checker is sound and enforced, so most of the old silent failures are now compile errors: Int/Float mixing, `(+ 1 "a")`, non-`Bool` conditions, a one-armed `if` used as a value, `<` on records, strings compared by address, `(let ((x 1) (y 2)) ...)`, nested patterns, guards outside literal arms, run-time trait dispatch, malformed special forms (which used to compile to 0), unknown string escapes, spawn entries with parameters, `Float` across the FFI. They are listed at the end so old habits can be checked. What remains silent:

## Compiles, computes wrong / nothing

| # | Pitfall | Rule |
|---|---|---|
| 1 | Integer literal ≥ 2^63 (e.g. `0xFF51AFD7ED558CCD`) becomes `0` | [syn-int-literal-range](../rules/syn-int-literal-range.md) |
| 2 | `\0` in a string literal ends the string (`"a\0b"` has length 1) | [syn-string-literals](../rules/syn-string-literals.md) |
| 3 | Misspelled constructor in the **last** match arm is a catch-all binding | [match-misspelled-last-arm](../rules/match-misspelled-last-arm.md) |
| 4 | Mixed literal + constructor arms: undiagnosed (`(match n (1 ...) (Red ...) (_ ...))` compiles) | [match-literal-requires-underscore](../rules/match-literal-requires-underscore.md) |
| 5 | Misspelled or unknown type name in an annotation (`(a Intt)`, an `alias` name) is a fresh type parameter: accepted, checks nothing | [type-sound-checking](../rules/type-sound-checking.md) |
| 6 | `print` of a struct/ADT with no `Show` impl, or of an `Option`/`List` of one, prints an address; `print` of a `Bool` prints `1`/`0` | [trait-derive-show](../rules/trait-derive-show.md) |
| 7 | Overflow wraps; `/` by zero → SIGFPE, exit 136, not catchable (REPL reports `E_DIVISION_BY_ZERO` instead) | [fn-integer-arith-unchecked](../rules/fn-integer-arith-unchecked.md) |
| 8 | Prelude `when`/`unless` evaluate the body even when the condition says not to | [fn-conditionals](../rules/fn-conditionals.md) |
| 9 | `with-resource` runs no cleanup; `alias` makes a type variable | [fn-unlowered-forms](../rules/fn-unlowered-forms.md) |
| 10 | `recover` tries arms in order and a `(String)`/`_` arm matches any error, so it shadows later `E_` arms; `checkpoint` does not undo byte-buffer writes; `--contracts=off`/`production` compiles every check out | [contract-checks-and-profiles](../rules/contract-checks-and-profiles.md) |
| 11 | `try` does not catch `Err` values (it catches panics) | [err-try-catches-error-not-err](../rules/err-try-catches-error-not-err.md) |
| 12 | Rebuilt record with two same-typed fields swapped (different types are now a type error) | [data-reconstruct-field-order](../rules/data-reconstruct-field-order.md) |
| 13 | Collection result discarded, or old version reused after update | [data-collections-persistent](../rules/data-collections-persistent.md) |
| 14 | `alias`, `(module …)`, `(export …)` do nothing (`derive` and `defstruct+ :derive` work) | [trait-derive-show](../rules/trait-derive-show.md) |
| 15 | Macro argument used twice is evaluated twice | [macro-args-spliced](../rules/macro-args-spliced.md) |
| 16 | `ffi-call` without a positive literal timeout is `E_FFI_TIMEOUT_REQUIRED`; `(ffi-call "f" 5)` is zero args + 5 ms; a too-tight timeout raises `E_FFI_TIMEOUT` and abandons the C call | [ffi-timeout-always-last](../rules/ffi-timeout-always-last.md) |
| 17 | `(ffi-pin str)` passed where C wants `const char*` | [ffi-pin-passes-pointer](../rules/ffi-pin-passes-pointer.md) |
| 18 | `receive` is untyped: a message of another type is read as whatever the receiver expects | [actor-send-is-discarded](../rules/actor-send-is-discarded.md) |
| 19 | `send` to an actor that never calls `(receive)` is dropped; `(receive)` on `main` with no sender hangs | [actor-send-is-discarded](../rules/actor-send-is-discarded.md) |
| 20 | A reply during the final drain can be lost | [actor-no-closure-messages](../rules/actor-no-closure-messages.md) |
| 21 | `main` returns without waiting: actor work killed | [actor-always-wait](../rules/actor-always-wait.md) |
| 22 | `shr` vs `ashr` confusion on bit patterns | [bits-shr-vs-ashr](../rules/bits-shr-vs-ashr.md) |
| 23 | Out-of-range byte load/store returns 0 silently | [bits-bounds-fail-closed](../rules/bits-bounds-fail-closed.md) |
| 24 | Unannotated helper launders a `Secret` (`set!` into a `let-mut` no longer does); heap copies of keys are never wiped automatically (only frames are) | [secret-unannotated-helpers-launder](../rules/secret-unannotated-helpers-launder.md), [secret-zeroize](../rules/secret-zeroize.md) |
| 25 | `test-suite` drops its tests; `assert-fail` always passes | [test-unimplemented-features](../rules/test-unimplemented-features.md) |
| 26 | Test binary exit status 0 despite failures | [test-read-summary-line](../rules/test-read-summary-line.md) |
| 27 | `--filter X` without `--full` runs nothing | [test-regression-runner](../rules/test-regression-runner.md) |
| 28 | Capability violations in `main` / top-level tests / manifest-less files unchecked | [pkg-capabilities](../rules/pkg-capabilities.md) |
| 29 | Stale `~/.zyl` shadows checkout stdlib | [pkg-stdlib-resolution](../rules/pkg-stdlib-resolution.md) |
| 30 | Unknown CLI word after the source becomes the output file name | [tool-cli-arguments](../rules/tool-cli-arguments.md) |
| 31 | `buf-append` on a non-fresh buffer accumulates | [proj-buf-append-appends](../rules/proj-buf-append-appends.md) |
| 32 | `with-region` limits (`E_REGION_EXHAUSTED`) are enforced only in compiled code; `zyl eval` and the REPL ignore regions | [own-with-region](../rules/own-with-region.md) |
| 33 | A slice shares its Vec's storage: a later `vec-set` through a Vec that still uses that storage shows through the slice | [data-collections-persistent](../rules/data-collections-persistent.md) |

## Compiler-contributor extras

| # | Pitfall | Rule |
|---|---|---|
| 34 | A missing closer swallows the rest of its file | [boot-parens-per-file](../rules/boot-parens-per-file.md) |
| 35 | Same type name in two modules: the later `use` wins, the other module misreads tags | [boot-one-deftype-per-name](../rules/boot-one-deftype-per-name.md) |
| 36 | Allocating inside a per-element lookup: compiler temporaries mostly land in the heap, which never frees | [pass-no-allocation-in-lookups](../rules/pass-no-allocation-in-lookups.md) |
| 37 | New special form without an `ic-expr-node` case lowers to 0 (a form whose parser rejects it is `E_MALFORMED_FORM`, but a parsed, unlowered one is still silent) | [icnf-new-form-needs-case](../rules/icnf-new-form-needs-case.md) |
| 38 | Register clobbering. Stack-machine code may touch only `rbx` and `r12` of the callee-saved set (every prologue saves exactly those); the native backend allocates `rsi rdi r8 r9 r10` and `rbx r12`–`r15` and saves only the callee-saved ones it used, so an emitted MIR sequence may clobber only the scratch registers `rax rcx rdx r11` | [cg-callee-saved-registers](../rules/cg-callee-saved-registers.md) |
| 39 | Rewritten node without `zyl_span_copy` loses locations | [pass-copy-spans](../rules/pass-copy-spans.md) |
| 40 | stdout writes corrupt the LSP channel | [pass-no-stdout](../rules/pass-no-stdout.md) |
| 41 | Listing a runtime function in `rg-ffi-kind` that keeps or aliases an argument (or allocates outside `zyl_result_alloc`): silent use-after-free once its region is released; the reuse pass trusts the same region summaries, so a wrong summary also lets it overwrite a live block | [icnf-regions-are-a-rewrite](../rules/icnf-regions-are-a-rewrite.md) |
| 42 | A new runtime entry called through `ffi-call` needs a signature in `ffi_sigs.zyl` (else `E_CANNOT_INFER`), and one that reads raw memory or reinterprets a word belongs in `ffi-raw-p` | [pass-diagnostics](../rules/pass-diagnostics.md) |

## Fixed: now compile errors (check old habits against these)

| Was silent | Now |
|---|---|
| `read-line`, `exit`, `close` lowered to 0 | lowered: stdin line, process exit, `file-close` ([fn-unlowered-forms](../rules/fn-unlowered-forms.md)) |
| lowercase `deftype` field type left the field unchecked | `E_UNKNOWN_TYPE` |
| own nullary constructor in a binder slot bound a variable | `E_NESTED_PATTERN` |
| a form after an `if`'s else branch was dropped | `E_MALFORMED_FORM` |
| a comma in an import list (a trailing one swallowed `main`) | `E_MALFORMED_FORM` |
| an extern'd `zyl_*` C function ran with no timeout | timed like any foreign call |
| stray `'` `` ` `` `,` `@` `#` truncated the file | `'` `` ` `` `,` `,@` are quote/quasiquote/unquote tokens (a stray `,` is `E_MALFORMED_FORM`); `@ # $ \| ^ \` are `E_INVALID_CHAR` ([syn-no-stray-characters](../rules/syn-no-stray-characters.md)) |
| unknown string escape → empty string | `E_INVALID_ESCAPE` |
| nested pattern tested only the outer tag; guard on a constructor arm crashed, on `_` was ignored | `E_NESTED_PATTERN`; `_`/`range` guards `E_ARITY_MISMATCH` ([match-no-nested-patterns](../rules/match-no-nested-patterns.md), [match-guards-literal-arms-only](../rules/match-guards-literal-arms-only.md)) |
| shared variant name disabled exhaustiveness | the later type owns the name: using it for the earlier type is `E_TYPE_MISMATCH`, a missed variant `E_MATCH_NONEXHAUSTIVE` ([data-unique-variant-names](../rules/data-unique-variant-names.md)) |
| `print`/`=` on an unsettled String/Float printed an address; `==` on strings compared pointers | every type is known; strings print and compare by content ([fn-types-drive-codegen](../rules/fn-types-drive-codegen.md), [fn-string-equality](../rules/fn-string-equality.md)) |
| Int/Float mixing, `(+ 1 "a")`, any type error | `E_TYPE_MISMATCH` ([fn-int-float-separation](../rules/fn-int-float-separation.md), [type-sound-checking](../rules/type-sound-checking.md)) |
| `(+ x)`, `(* x)`, `(/ x)` evaluated to 0 | `(+ x)`/`(* x)` are `x`; `(/ x)` `(% x)` are `E_ARITY_MISMATCH` |
| `let` among body forms leaked scope; `(let (x 1) a b)` dropped `b`; `(let ((x 1) (y 2)) …)` was a wrong program | bodies are implicit `begin`s; the binding list is `E_MALFORMED_FORM` ([fn-let-single-binding](../rules/fn-let-single-binding.md), [fn-begin-multi-form-bodies](../rules/fn-begin-multi-form-bodies.md)) |
| one-armed `if` / `cond` without `else` → 0 | they are `Unit`; using one as a value is `E_TYPE_MISMATCH` ([fn-conditionals](../rules/fn-conditionals.md)) |
| `make-variant` → 0 | `E_CANNOT_INFER` / `E_MALFORMED_FORM` |
| `<`/`>` on records ordered by address | `E_TYPE_MISMATCH`; derive `Ord`, call `Ord.compare` ([data-equality-structural](../rules/data-equality-structural.md)) |
| `((T : Ord) a b)` added a value parameter; `(a : Int)` ignored the type | `E_MALFORMED_PARAMETER` ([gen-no-type-parameter-syntax](../rules/gen-no-type-parameter-syntax.md)) |
| trait call on mixed-type data used a run-time fallback | no run-time dispatch: `E_TRAIT_NOT_FOUND` / `E_CANNOT_INFER` ([trait-static-dispatch](../rules/trait-static-dispatch.md)) |
| macro body kept only its last form | `E_MALFORMED_FORM` ([macro-args-spliced](../rules/macro-args-spliced.md)) |
| Float passed to C `double` | `Float` in an `extern` is `E_TYPE_MISMATCH` ([ffi-extern-word-sized-types](../rules/ffi-extern-word-sized-types.md)) |
| spawned `fn` parameter was 0 | a spawn entry with parameters is `E_TYPE_MISMATCH` ([actor-spawn-zero-arg-entry](../rules/actor-spawn-zero-arg-entry.md)) |
| `vec-get` out of range returned word -1 | `E_INDEX_OUT_OF_BOUNDS` panic (`vec-get-or` for a default) |
| `error` in a 2/4-arg function under `try` hung (2026-09-24) | fixed ([err-try-any-arity](../rules/err-try-any-arity.md)) |
| `=` on dynamic strings in the compiler compared pointers | content comparison ([pass-string-eq-in-compiler](../rules/pass-string-eq-in-compiler.md); `str-eq` remains the convention) |
