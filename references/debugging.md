# Debugging: Symptom → Cause → Fix

## Programs

| Symptom | Likely cause | Fix |
|---|---|---|
| Function missing from output / `E_UNBOUND_VARIABLE` for a visible function | paren imbalance nesting it in an earlier form | check balance of preceding forms |
| A list of `error[E_TYPE_MISMATCH]` / `E_CANNOT_INFER` ending in `the program does not type-check (N errors above)` | the type checker reports every error, then fails; the first is usually the cause of the rest | fix from the top; `ZYL_DEBUG_TYPES=1` prints each function's inferred scheme on stderr |
| `cannot unify Int with Unit` at `main` or a `defn` | `main` must return an `Int` (end with `0`); a body ending in `print`/`set!`/one-armed `if` is `Unit` | end with the value |
| `cannot unify Int with Bool` | an `Int` used as a condition, or `and`/`or`/`not` on Ints | compare: `(= n 0)` |
| `cannot unify Int with Float` | `Int` and `Float` mixed in arithmetic | write the literal in the right type; `(ffi-call "zyl_f_of_int" n 1000)` |
| `E_CANNOT_INFER: no type for ffi-call to `f`` | a foreign symbol with no `extern` | `(extern "f" (Int) Int)` |
| `E_CANNOT_INFER: no type for untyped ffi result` | a `zyl_*` runtime entry with no signature in `ffi_sigs.zyl` (such as `zyl_actor_send_closure`) | use the typed form or stdlib function; programs cannot call it |
| `E_CANNOT_INFER` on a byte load/store | handle could be `ByteBuf` or `ByteSlice` | annotate `((b ByteBuf))` |
| `E_FFI_RESTRICTED` | a raw runtime entry (`zyl_word_*`, `zyl_view_*`, `zyl_mem_read`, ...) in user code, or an `extern` for a `zyl_*` entry | call the stdlib function built on it; drop the `extern` |
| `E_MALFORMED_FORM` | a special form of the wrong shape (binding-list `let`, `if` without branches, several forms in a `test`/`defmacro`), a name in quoted data, a stray `,`/`,@` | see the form's shape in [builtins.md](builtins.md) |
| `E_MALFORMED_PARAMETER` on `(a : Int)` or `(a Ord)` | colon annotation, or a trait used as a type | `(a Int)`; an unannotated parameter for trait-generic code |
| `error[E_INVALID_CHAR]` | `@` `#` `$` `\|` `^` `\`, a lone `.`, non-ASCII or a BOM outside strings/comments (an old compiler instead truncated the file silently, giving `undefined reference to _ZYL_main`) | [syn-no-stray-characters](../rules/syn-no-stray-characters.md) |
| Large integer printed instead of a value | a struct/ADT (or an `Option`/`List` of one) with no `Show` impl | `(derive T Show)` or an `impl Show` |
| `1`/`0` printed for a Bool | `print` shows `Bool` as 1/0 | print `(if b "true" "false")` |
| Value is 0 unexpectedly | oversized literal, out-of-range byte load, `print`'s `Unit` result printed | see [pitfalls](pitfalls.md) |
| Match arm never taken / always taken | misspelled constructor (last arm catch-all); missing `use` of the defining module | [match-misspelled-last-arm](../rules/match-misspelled-last-arm.md) |
| `E_UNREACHABLE_MATCH_ARM` right after `Nil`/`None` in a module | constructor unknown there | add `(use core/list)` etc. |
| `E_NESTED_PATTERN` | a constructor field written as a pattern, or a guard on a constructor arm | bind a name, match inside the body |
| `E_ARITY_MISMATCH: `when` called with 1 argument` in a match | guard on a `range` or `_` arm | guard only literal arms, or test in the body |
| Garbage where a variable should be | reconstructed record with same-typed fields out of order | [data-reconstruct-field-order](../rules/data-reconstruct-field-order.md) |
| SIGFPE (exit 136) | integer `/` or `%` by zero | guard divisor |
| `PANIC: E_INDEX_OUT_OF_BOUNDS` | `vec-get`/`vec-last`/`slice-*` outside the range | `vec-get-or`, check `vec-len` |
| Infinite loop | `for` without `set!` of the loop var | [fn-for-has-no-step](../rules/fn-for-has-no-step.md) |
| `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` with no `main` of yours | a `use`d module defines `main` | check the transitive `use` chain |
| `E_UNBOUND_VARIABLE` for a trait method | bare method call `(method r)` | qualify: `(Trait.method r)` or `(r.method)` |
| `E_TRAIT_NOT_FOUND ... no impl of `Tr.m` for type `X`` | no impl for that type (located at the call) | add `(impl Tr X ...)` or `(derive X Tr)` |
| `E_TRAIT_NOT_DERIVABLE` | a field type lacks the trait, a `Secret` field under Eq/Ord/Hash, or an underivable name | derive the field types first; drop the trait |
| `E_DUPLICATE_IMPL` | two `(impl Tr X ...)`, or `Tr` derived twice for `X` | keep one |
| Actor output missing | `main` returned first | `actor-wait` / `(ffi-call "zyl_actor_wait_all" 1000)` |
| Actor reads a nonsense message | `receive` is untyped; a sender sent another type | one message type per actor |
| C function gets garbage | pinned string passed where C expects a `char*`; an `extern` that does not match the C prototype | [ffi-pin-passes-pointer](../rules/ffi-pin-passes-pointer.md), [ffi-extern-word-sized-types](../rules/ffi-extern-word-sized-types.md) |
| `zyl eval` and binary disagree | codegen or interpreter bug (shared front end) | bisect with prefixes + canary; `ZYL_MIR=0` / `ZYL_REUSE=0` / `ZYL_INLINE=0` to isolate a backend pass; interpreter test category |
| Edits to stdlib/compiler have no effect outside the checkout | stale `~/.zyl` wins resolution | `ZYL_HOME=$PWD/build/boot` or `./install.sh` |
| Test run "passes" but did nothing | `test-suite` wrapper, missing `(run-tests)`, `--filter` without `--full` | flat tests; read summary |
| Exit 139 / 134 | compiler/runtime bug / internal abort | minimize and report |

## Compiler / bootstrap

| Symptom | Likely cause | Fix |
|---|---|---|
| `reproduced asm differs from committed seed` | compiler output changed | reseed ([boot-fixed-point-workflow](../rules/boot-fixed-point-workflow.md)) |
| `FIXED POINT BROKEN` | non-determinism or pending reseed | [boot-failure-modes](../rules/boot-failure-modes.md) |
| The compiler's own source no longer type-checks after an edit | the compiler is checked like any program | fix the type error; never weaken the checker |
| Count/sum is 0 or too small in stage ≥ 2 | historical two-call binop, or record fields out of order | pre-bind calls with `let`; check field order |
| Output truncated to last emitted line | copy instead of append in emission | `zyl_str_append` / `buf-append` |
| `E_UNBOUND_VARIABLE` for another module's function | missing `use` | add `use` (the help line may suggest the name you meant) |
| `E_OUT_OF_MEMORY` during `./boot.sh` | a compiler change allocates far more (the cap is 4 GB per stage, `ZYL_STAGE_MEMORY`) | [pass-no-allocation-in-lookups](../rules/pass-no-allocation-in-lookups.md) |
| C caller, or a native-backend caller, misbehaves after calling a function | callee-saved register clobbered: a stack-machine sequence touched `r13`–`r15`, or a MIR emission sequence clobbered an allocated register (`rsi rdi r8`–`r10`, `rbx r12`–`r15`) instead of scratch `rax rcx rdx r11` | [cg-callee-saved-registers](../rules/cg-callee-saved-registers.md); bisect with `ZYL_MIR=0` |
| Wrong result only with the native backend | MIR lowering or register allocation bug | `ZYL_MIR=0` at compile time; if that fixes it, the function was `mb-eligible` |
| Wrong result only with reuse | a value treated as unique and dead is still referenced | `ZYL_REUSE=0`; `ZYL_REUSE_DEBUG=1` prints each function's reuse facts |
| Wrong result only with inlining | inlined body captured a name or changed a kind | `ZYL_INLINE=0`; `ZYL_INLINE_LIMIT=N` to narrow |
| `movaps` fault in libc | unaligned C call | `cg-ext-call-aligned` |
| Diagnostics lose location after a new pass | spans not copied | `zyl_span_copy` |
| Compile time explodes with nesting | pass re-walks subtrees | visit once |
| New form evaluates to 0 | no `ic-expr-node` case | add lowering |

## Tools

- `zyl prog.zyl --emit-asm -o prog.s` (only intermediate output; labels are mangled canonical keys).
- `ZYL_DEBUG_STAGES=1 zyl prog.zyl -o prog` → stage names appended to `/tmp/dbg`; last line = crashing/hanging stage. (`cg-dbg` in codegen appends to `/tmp/dbg2` and has no callers.)
- `ZYL_DEBUG_TYPES=1` → every function's inferred type scheme on stderr; `ZYL_STRICT_TYPES=report` → type errors as `W_TYPE_STRICT` warnings, compile continues (for counting only).
- Backend switches, read at compile time: `ZYL_MIR=0` (stack machine only), `ZYL_INLINE=0`, `ZYL_INLINE_LIMIT=N` (default 6), `ZYL_REUSE=0`, `ZYL_REUSE_DEBUG=1`, `ZYL_REGIONS=0` (every allocation on the heap).
- `ZYL_INTERP_CHECK=1 zyl eval prog.zyl` → the interpreter checks every operand's tag (`E_INTERP_TAG` means a type-checker bug).
- Log an integer from compiler code: `(ffi-call "zyl_int_text" n 1000)` → String.
- `zyl eval prog.zyl` vs `./prog`.
- `cc -g -no-pie prog.s ~/.zyl/actor_runtime.c -o prog -lpthread && gdb ./prog`.
- Small driver programs that `use` `compiler/*` modules to inspect ASTs / ICNF.
- REPL `:type EXPR`, `:time EXPR`.
