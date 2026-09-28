# boot-two-step-syntax

> Introduce new syntax (and any new runtime function the compiler calls) in two steps: teach the compiler to accept it and reseed, then start using it in the compiler's own source.

## Why It Matters

`--bootstrap-from-self` fails only when the old seed cannot even **parse** the new source (new syntax, not new behavior). There is no fallback compiler: the Rust bootstrap was removed (git history at `b8bc283`) and cannot lex the current source.

## Steps

1. Add lexer/parser/lowering support for the new form; do **not** use it anywhere in `stdlib/compiler/`, `selfhost/`, `stdlib/repl/`, `stdlib/lsp/builtins.zyl` or anything else `selfhost/driver.zyl` reaches through `use`.
2. `./boot.sh --bootstrap-from-self && ./boot.sh`; commit the seed.
3. Now use the syntax in compiler source; reseed again.

## Notes

- The seed also type-checks the new source with its own type pass, whose catch-all makes a form it has no case for `E_CANNOT_INFER` (`no type for form not typed`). A new special form therefore needs the seed to know how to type it, not only how to parse it.
- The same applies to new built-ins the compiler itself wants to call, to new escapes in string literals, and to new runtime functions: the seed types every `ffi-call` in the compiler by its own `ffi_sigs.zyl`, so add the `zyl_*` defn in `runtime/rt/` and its signature first, reseed, then call it ([boot-fixed-point-workflow](boot-fixed-point-workflow.md)). On 2026-09-28 `zyl_actor_id`, `zyl_chan_spawn_moves` and `zyl_actor_join_unjoined` landed this way before the REPL called them.
- History: the bitwise operators were once unusable in compiler source because the seed of the day could not compile them, which is why the optimizer does not fold them. That limitation is gone (`mir.zyl` uses `bit-and`); the folding was simply never added.

## See Also

- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
