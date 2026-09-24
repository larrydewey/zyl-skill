# Debugging: Symptom → Cause → Fix

## Programs

| Symptom | Likely cause | Fix |
|---|---|---|
| Function missing from output / `E_UNBOUND_VARIABLE` for a visible function | paren imbalance nesting it in an earlier form | check balance of preceding forms |
| `error[E_INVALID_CHAR]` | `'` `` ` `` `,` `@` `#` …, non-ASCII or a BOM outside strings/comments (an old compiler instead truncated the file silently, giving `undefined reference to _ZYL_main`) | [syn-no-stray-characters](../rules/syn-no-stray-characters.md) |
| Large integer printed instead of text | string whose type inference could not settle (mixed-type data, untyped FFI result, conflicting uses); or a record with no `Show` impl | `ZYL_DEBUG_TYPES=1` to see the inferred types; `print-string`; `(derive T Show)` |
| Huge integer instead of a float | float of conflicting/unknown type; Int and Float mixed | `ZYL_DEBUG_TYPES=1`; `print-float`; keep Int and Float apart |
| String comparison always false | `=` on strings of unknown type (address compare) | `str-eq` |
| Value is 0 unexpectedly | `make-variant`, `read-line`, `exit`, unary `(+ x)`, one-armed `if`, `cond` without else, unmatched match, oversized literal, out-of-range byte load, `print`'s return value | see [pitfalls](pitfalls.md) |
| Match arm never taken / always taken | misspelled constructor (last arm catch-all); nested pattern; shared variant name; missing `use` of the defining module | [match-misspelled-last-arm](../rules/match-misspelled-last-arm.md) |
| `E_UNREACHABLE_MATCH_ARM` right after `Nil`/`None` in a module | constructor unknown there | add `(use core/list)` etc. |
| Crash with a guard in a match | guard on a constructor arm | test in the body |
| Garbage where a variable should be | reconstructed record with fields out of order | [data-reconstruct-field-order](../rules/data-reconstruct-field-order.md) |
| SIGFPE | integer `/` or `%` by zero | guard divisor (compiler: `cqo` before `idiv`) |
| Infinite loop | `for` without `set!` of the loop var | [fn-for-has-no-step](../rules/fn-for-has-no-step.md) |
| `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` with no `main` of yours | a `use`d module defines `main` | check the transitive `use` chain |
| Link error `undefined reference` to a trait method | bare method call `(method r)` | qualify: `(Trait.method r)` or `(r.method)` |
| `E_TRAIT_NOT_FOUND ... no impl of `Tr.m` for type `X`` | no impl for that type (located at the call) | add `(impl Tr X ...)` or `(derive X Tr)` |
| `E_TRAIT_NOT_DERIVABLE` | a field type lacks the trait, a `Secret` field under Eq/Ord/Hash, or an underivable name | derive the field types first; drop the trait |
| `E_DUPLICATE_IMPL` | two `(impl Tr X ...)`, or `Tr` derived twice for `X` | keep one |
| Actor output missing | `main` returned first | `actor-wait` / `zyl_actor_wait_all` |
| Actor segfaults | spawned closure captures a variable | top-level entry function |
| C function gets garbage | missing ffi timeout; float argument; pinned string | [ffi-timeout-always-last](../rules/ffi-timeout-always-last.md) |
| `zyl eval` and binary disagree | codegen or interpreter bug (shared front end) | bisect with prefixes + canary; interpreter test category |
| Edits to stdlib/compiler have no effect outside the checkout | stale `~/.zyl` wins resolution | `ZYL_HOME=$PWD/build/boot` or `./install.sh` |
| Test run "passes" but did nothing | `test-suite` wrapper, missing `(run-tests)`, `--filter` without `--full` | flat tests; read summary |
| Exit 139 / 134 | compiler/runtime bug / internal abort | minimize and report |

## Compiler / bootstrap

| Symptom | Likely cause | Fix |
|---|---|---|
| `reproduced asm differs from committed seed` | compiler output changed | reseed ([boot-fixed-point-workflow](../rules/boot-fixed-point-workflow.md)) |
| `FIXED POINT BROKEN` | non-determinism or pending reseed | [boot-failure-modes](../rules/boot-failure-modes.md) |
| Count/sum is 0 or too small in stage ≥ 2 | historical two-call binop, or record fields out of order | pre-bind calls with `let`; check field order |
| Output truncated to last emitted line | copy instead of append in emission | `zyl_str_append` / `buf-append` |
| `E_UNBOUND_VARIABLE` for another module's function | missing `use` | add `use` (the help line may suggest the name you meant) |
| `E_OUT_OF_MEMORY` during `./boot.sh` | a compiler change allocates far more | [pass-no-allocation-in-lookups](../rules/pass-no-allocation-in-lookups.md) |
| C caller crashes after calling Zyl (qsort, tests, actors) | callee-saved register other than rbx/r12 clobbered | extend save/restore |
| `movaps` fault in libc | unaligned C call | `cg-ext-call-aligned` |
| Diagnostics lose location after a new pass | spans not copied | `zyl_span_copy` |
| Compile time explodes with nesting | pass re-walks subtrees | visit once |
| New form evaluates to 0 | no `ic-expr-node` case | add lowering |

## Tools

- `zyl prog.zyl --emit-asm -o prog.s` (only intermediate output; labels are mangled canonical keys).
- `ZYL_DEBUG_STAGES=1 zyl prog.zyl -o prog` → stage names appended to `/tmp/dbg`; last line = crashing/hanging stage. (`cg-dbg` in codegen appends to `/tmp/dbg2` and has no callers.)
- Log an integer from compiler code: `(ffi-call "zyl_cstr_from_int" arena n 1000)`.
- `zyl eval prog.zyl` vs `./prog`.
- `cc -g -no-pie prog.s ~/.zyl/actor_runtime.c -o prog -lpthread && gdb ./prog`.
- Small driver programs that `use` `compiler/*` modules to inspect ASTs / ICNF.
- REPL `:type EXPR`, `:time EXPR`.
