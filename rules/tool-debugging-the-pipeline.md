# tool-debugging-the-pipeline

> Debug with `--emit-asm`, `ZYL_DEBUG_STAGES=1`, `zyl eval` differential runs, and small driver programs that `use` compiler modules.

## Why It Matters

There is no phase dump beyond assembly and no ICNF printer, so you need the right tool per symptom.

| Issue | Tool |
|---|---|
| Unbalanced brackets | balance check prints line, column, fix-it |
| Resolver/capability/check failure | the `PANIC: E_...` message names the pass and qualified definition |
| Which stage fails or hangs | `ZYL_DEBUG_STAGES=1 zyl prog.zyl -o prog`; last line of `/tmp/dbg` |
| Wrong code | `zyl prog.zyl --emit-asm -o prog.s`; labels are mangled canonical keys |
| Compiled vs intended semantics | `zyl eval prog.zyl` vs `./prog` (differ ⇒ codegen or interpreter bug) |
| Link errors | misspelled function, private symbol via `*`, unlinked C symbol, trait call inside `struct-get` |
| Runtime crash | `cc -g -no-pie prog.s ~/.zyl/actor_runtime.c -o prog -lpthread && gdb ./prog` (no DWARF for Zyl code) |
| Exit 139 / 134 | segfault / abort: compiler bug or runtime internal error — report it |

## Notes

- Every check stops at its first error: one diagnostic per compile.
- Located diagnostics: `error[CODE]: msg`, `--> file:line:col`, source line, caret, `= help:`. Unlocated ones: `PANIC: CODE: ...`.
- For deeper symptom → cause mapping see [references/debugging.md](../references/debugging.md).

## See Also

- [tool-eval-differential](tool-eval-differential.md)
- [test-compiler-internals](test-compiler-internals.md)
