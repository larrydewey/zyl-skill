# tool-debugging-the-pipeline

> Debug with `--emit-asm`, the `ZYL_*` bisection switches (`ZYL_MIR=0`, `ZYL_INLINE=0`, `ZYL_REUSE=0`, `ZYL_REGIONS=0`), `ZYL_DEBUG_STAGES`/`ZYL_DEBUG_TYPES`, `zyl eval` differential runs, and small driver programs that `use` compiler modules.

## Why It Matters

There is no command-line phase dump beyond assembly (`compiler/icnf_print.zyl` prints ICNF, but only for the package-build hash or a driver program), so you need the right tool per symptom. The back end now has several optimizing layers, each with an off switch read at compile time; turning them off one at a time bisects a miscompile to its layer.

| Issue | Tool |
|---|---|
| Unbalanced brackets | balance check prints line, column, fix-it |
| Type errors | every one is reported, located, before the compile fails; `ZYL_DEBUG_TYPES=1` prints each function's inferred scheme (`noisy : ( String '1 -> '1)`) on stderr |
| Resolver/capability/check failure | the `error[E_...]` (or bare `PANIC: E_...`) message names the pass and qualified definition |
| Which stage fails or hangs | `ZYL_DEBUG_STAGES=1 zyl prog.zyl -o prog`; last line of `/tmp/dbg` (stages: balance-check, parse, modules, macros, the checks, lift-impls, closure-inline, lower (which includes type checking), optimize, region-infer, reuse, codegen) |
| Wrong code | `zyl prog.zyl --emit-asm -o prog.s`; labels are mangled canonical keys |
| Which layer miscompiles | rebuild with `ZYL_MIR=0` (stack-machine back end instead of MIR + register allocation), `ZYL_INLINE=0` (no inlining or copy propagation), `ZYL_REUSE=0` (no in-place reuse), `ZYL_REGIONS=0` (everything on the heap); the switch that fixes it names the layer |
| Reuse decisions | `ZYL_REUSE_DEBUG=1`: each function's facts and the fixpoint's round count, on stderr |
| Compiled vs intended semantics | `zyl eval prog.zyl` vs `./prog` (differ ⇒ codegen or interpreter bug); `ZYL_INTERP_CHECK=1 zyl eval` also checks operand tags |
| Link errors | an unlinked C symbol (a declared `extern` or `native` source missing); undefined Zyl functions are caught earlier, as `E_UNBOUND_VARIABLE` |
| Runtime crash | `cc -g -no-pie prog.s ~/.zyl/actor_runtime.c -o prog -lpthread && gdb ./prog` (no DWARF for Zyl code) |
| Exit 139 / 134 / 136 | segfault / abort / SIGFPE (integer division by zero in compiled code): a crash other than division by zero is a compiler bug or runtime internal error — report it |
| Performance | `bench/`: `./bench/build.sh`, then `python3 bench/matrix.py` (seven programs in Zyl, C, C++, Rust and Go; best of three wall time and peak RSS, `(!)` when outputs differ) |

## Notes

- The checks before type checking (duplicate, arity, mutability, exhaustiveness, unused, secret) stop at their first error; the type checker reports every type error, then fails.
- Located diagnostics: `error[CODE]: msg` (or `warning[CODE]`), `--> file:line:col`, source line, caret, optional labelled secondary spans (`::: file:line:col` with `- parameter ... declared here`), `= help:`. Unlocated ones: `PANIC: CODE: ...`. `--error-format=json` prints either as one JSON object per line on stderr.
- `ZYL_STRICT_TYPES=report` turns type errors into `W_TYPE_STRICT` warnings and continues: useful to count errors in a large port, never to run the result.
- The switches affect only the compile that reads them, and the output of a correct program must not change with any of them.
- For deeper symptom → cause mapping see [references/debugging.md](../references/debugging.md).

## See Also

- [tool-eval-differential](tool-eval-differential.md)
- [test-compiler-internals](test-compiler-internals.md)
- [tool-cli-arguments](tool-cli-arguments.md)
