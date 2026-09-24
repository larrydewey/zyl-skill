# tool-eval-differential

> Use `zyl eval` / the REPL for fast iteration, but confirm behavior with a compiled binary: the interpreter differs on actors, FFI, division by zero and speed.

## Why It Matters

`zyl eval` and `zyl repl` run the same front end (`compile-to-fns`) and then interpret ICNF in-process. Differences:

| | Compiled | Interpreter |
|---|---|---|
| Actors | yes | `E_UNSUPPORTED_INTERPRETED` |
| FFI | any linked symbol, any arity | `dlsym`, ≤ 6 args, `E_FFI_SYMBOL_NOT_FOUND` |
| Integer division by zero | SIGFPE | `E_DIVISION_BY_ZERO` |
| Undefined function | link error | `E_UNDEFINED_FUNCTION` |
| Heavy arithmetic | fast | allocates per op; can be 1000× slower and hit `E_OUT_OF_MEMORY` |
| Values printed | per kind (structs as addresses) | structurally |

Because everything up to ICNF is shared, compiled-vs-eval disagreement localizes a bug to codegen (or the interpreter). The `interpreter` test category diffs the two over every regression and smoke test.

## See Also

- [tool-repl](tool-repl.md)
- [fn-integer-arith-unchecked](fn-integer-arith-unchecked.md)
