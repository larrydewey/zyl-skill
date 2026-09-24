# Silent-Failure Checklist

Things that compile without complaint and then do the wrong thing. Scan this before shipping or reviewing Zyl code. Each links to the rule with the fix.

## Compiles, computes wrong / nothing

| # | Pitfall | Rule |
|---|---|---|
| 1 | Stray `'` `` ` `` `,` `@` `#` `|` … outside strings truncates the rest of the file | [syn-no-stray-characters](../rules/syn-no-stray-characters.md) |
| 2 | Integer literal ≥ 2^63 (e.g. `0xFF51AFD7ED558CCD`) becomes `0` | [syn-int-literal-range](../rules/syn-int-literal-range.md) |
| 3 | Unknown string escape → empty string; `\0` truncates | [syn-string-literals](../rules/syn-string-literals.md) |
| 4 | Misspelled constructor in the **last** match arm is a catch-all | [match-misspelled-last-arm](../rules/match-misspelled-last-arm.md) |
| 5 | Nested pattern `(Some (Cons x _))`: inner constructor never tested | [match-no-nested-patterns](../rules/match-no-nested-patterns.md) |
| 6 | Guard on a constructor arm compiles then crashes; guard on `_` ignored | [match-guards-literal-arms-only](../rules/match-guards-literal-arms-only.md) |
| 7 | Mixed literal + constructor arms: undiagnosed | [match-literal-requires-underscore](../rules/match-literal-requires-underscore.md) |
| 8 | Shared variant name across types disables exhaustiveness | [data-unique-variant-names](../rules/data-unique-variant-names.md) |
| 9 | `print`/`=` on unannotated String/Float params, fields, captures, polymorphic results → addresses/bit patterns | [fn-annotate-string-float-params](../rules/fn-annotate-string-float-params.md) |
| 10 | `==` on strings of unknown kind compares addresses | [fn-string-equality](../rules/fn-string-equality.md) |
| 11 | Int/Float mixing computes garbage | [fn-int-float-separation](../rules/fn-int-float-separation.md) |
| 12 | `(+ x)`, `(* x)`, `(/ x)` evaluate to 0 | [fn-integer-arith-unchecked](../rules/fn-integer-arith-unchecked.md) |
| 13 | Overflow wraps; `/` by zero → SIGFPE (REPL reports it instead) | [fn-integer-arith-unchecked](../rules/fn-integer-arith-unchecked.md) |
| 14 | `let` among several body forms without `begin` leaks scope | [fn-begin-multi-form-bodies](../rules/fn-begin-multi-form-bodies.md) |
| 15 | `(let (x 1) a b)` drops `b`; `(let ((x 1) (y 2)) …)` wrong program | [fn-let-single-binding](../rules/fn-let-single-binding.md) |
| 16 | One-armed `if` / `cond` without `else` → 0 | [fn-conditionals](../rules/fn-conditionals.md) |
| 17 | Prelude `when`/`unless` evaluate the body even when false | [fn-conditionals](../rules/fn-conditionals.md) |
| 18 | `assert` no-op, `unwrap` → 0, `read-line`/`exit`/`close` not lowered | [fn-unlowered-forms](../rules/fn-unlowered-forms.md) |
| 19 | Contracts evaluated and ignored | [contract-not-enforced](../rules/contract-not-enforced.md) |
| 20 | `try` does not catch `Err` values | [err-try-catches-error-not-err](../rules/err-try-catches-error-not-err.md) |
| 21 | `error` in a 2/4-arg function under `try` can hang | [err-try-even-arity-hang](../rules/err-try-even-arity-hang.md) |
| 22 | Rebuilt record with swapped fields | [data-reconstruct-field-order](../rules/data-reconstruct-field-order.md) |
| 23 | Collection result discarded, or old version reused after update | [data-collections-int-persistent](../rules/data-collections-int-persistent.md) |
| 24 | Shallow `==`/`assert-equal` on nested data | [data-equality-shallow](../rules/data-equality-shallow.md) |
| 25 | Type errors (`(+ 1 "a")`, wrong annotation) accepted | [type-inference-does-not-reject](../rules/type-inference-does-not-reject.md) |
| 26 | `((T : Ord) a b)` adds a value parameter | [gen-no-type-parameter-syntax](../rules/gen-no-type-parameter-syntax.md) |
| 27 | Multiple trait impls on ADTs/primitives dispatch to the wrong impl | [trait-dispatch-structs-only](../rules/trait-dispatch-structs-only.md) |
| 28 | `derive`, `alias`, `defstruct+ :derive`, `(module …)`, `(export …)` do nothing | [trait-derive-noop](../rules/trait-derive-noop.md) |
| 29 | Macro argument used twice is evaluated twice; macro body keeps only its last form | [macro-args-spliced](../rules/macro-args-spliced.md) |
| 30 | `ffi-call` without timeout drops the last real argument | [ffi-timeout-always-last](../rules/ffi-timeout-always-last.md) |
| 31 | Float passed to C `double` | [ffi-int64-only-no-floats](../rules/ffi-int64-only-no-floats.md) |
| 32 | `(ffi-pin str)` passed where C wants `const char*` | [ffi-pin-passes-pointer](../rules/ffi-pin-passes-pointer.md) |
| 33 | `send` messages discarded; no `receive` | [actor-send-is-discarded](../rules/actor-send-is-discarded.md) |
| 34 | Spawned closure capturing anything crashes | [actor-spawn-captures-nothing](../rules/actor-spawn-captures-nothing.md) |
| 35 | `main` returns without waiting: actor work killed | [actor-always-wait](../rules/actor-always-wait.md) |
| 36 | `shr` vs `ashr` confusion on bit patterns | [bits-shr-vs-ashr](../rules/bits-shr-vs-ashr.md) |
| 37 | Out-of-range byte load/store returns 0 silently | [bits-bounds-fail-closed](../rules/bits-bounds-fail-closed.md) |
| 38 | Unannotated helper launders a `Secret` | [secret-unannotated-helpers-launder](../rules/secret-unannotated-helpers-launder.md) |
| 39 | `test-suite` drops its tests; `assert-fail` always passes | [test-unimplemented-features](../rules/test-unimplemented-features.md) |
| 40 | Test binary exit status 0 despite failures | [test-read-summary-line](../rules/test-read-summary-line.md) |
| 41 | `--filter X` without `--full` runs nothing | [test-regression-runner](../rules/test-regression-runner.md) |
| 42 | Capability violations in `main` / top-level tests / manifest-less files unchecked | [pkg-capabilities](../rules/pkg-capabilities.md) |
| 43 | Stale `~/.zyl` shadows checkout stdlib | [pkg-stdlib-resolution](../rules/pkg-stdlib-resolution.md) |
| 44 | Unknown CLI word after the source becomes the output file name | [tool-cli-arguments](../rules/tool-cli-arguments.md) |
| 45 | `buf-append` on a non-fresh buffer accumulates | [proj-buf-append-appends](../rules/proj-buf-append-appends.md) |

## Compiler-contributor extras

| # | Pitfall | Rule |
|---|---|---|
| 46 | Per-file paren deficit masked in the bundle | [boot-parens-per-file](../rules/boot-parens-per-file.md) |
| 47 | Duplicate `defn` in the bundle: first wins, second never runs | [boot-assemble-bundle](../rules/boot-assemble-bundle.md) |
| 48 | Missing `use` invisible in the bundle, link error standalone | [pkg-use-what-you-construct](../rules/pkg-use-what-you-construct.md) |
| 49 | New special form without an `ic-expr-node` case lowers to 0 | [icnf-new-form-needs-case](../rules/icnf-new-form-needs-case.md) |
| 50 | New codegen sequence clobbers `r13`–`r15` | [cg-callee-saved-rbx-r12](../rules/cg-callee-saved-rbx-r12.md) |
| 51 | Rewritten node without `zyl_span_copy` loses locations | [pass-copy-spans](../rules/pass-copy-spans.md) |
| 52 | `=` on dynamic strings in the compiler (pointer compare, fixed-point risk) | [pass-string-eq-in-compiler](../rules/pass-string-eq-in-compiler.md) |
| 53 | stdout writes corrupt the LSP channel | [pass-no-stdout](../rules/pass-no-stdout.md) |
