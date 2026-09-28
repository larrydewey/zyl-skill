# tool-eval-differential

> Use `zyl eval` / the REPL for fast iteration, but confirm behavior with a compiled binary: the interpreter differs on FFI, division by zero, regions and speed.

## Why It Matters

`zyl eval` and `zyl repl` run the same front end (`compile-to-fns`) and then interpret ICNF in-process. Differences:

| | Compiled | Interpreter |
|---|---|---|
| Actors | threads, channels | the same: a spawn runs a compiled closure that interprets the body; output and schedules match |
| `with-region` limits | enforced (`E_REGION_EXHAUSTED`) | ignored: the interpreter allocates in its own arenas |
| FFI | any linked symbol, up to 16 args, timeout enforced | `dlsym`, foreign calls through `zyl_ffi_timed_argv` (timeout enforced, ≤ 16 args), `E_FFI_SYMBOL_NOT_FOUND` |
| Integer division by zero | SIGFPE (exit 136) | `E_DIVISION_BY_ZERO` (exit 1) |
| Undefined function | `E_UNBOUND_VARIABLE` from the type checker, before either runs | same |
| Heavy arithmetic | fast | allocates per op; can be 1000× slower and hit `E_OUT_OF_MEMORY` |
| Values printed | by `Show`; a struct or ADT without one prints its address | the same (the address differs) |

`compile-to-fns` runs every phase before codegen (type checking, inlining, region inference, reuse), so all of that is shared; the interpreter then ignores the region annotations and reuse decisions. A compiled-vs-eval disagreement therefore localizes a bug to codegen, to what a region or reuse decision does at run time, or to the interpreter: rebuild with `ZYL_REGIONS=0`, `ZYL_REUSE=0` or `ZYL_MIR=0` to tell which. `ZYL_INTERP_CHECK=1 zyl eval prog.zyl` adds the checking mode: every operator checks its operand tags and every condition must be 0 or 1. The `interpreter` test category runs every regression and smoke test that way and diffs the output with the binary's.

## See Also

- [tool-repl](tool-repl.md)
- [fn-integer-arith-unchecked](fn-integer-arith-unchecked.md)
