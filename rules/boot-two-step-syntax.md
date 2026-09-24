# boot-two-step-syntax

> Introduce new syntax in two steps: teach the compiler to accept it and reseed, then start using it in the compiler's own source.

## Why It Matters

`--bootstrap-from-self` fails only when the old seed cannot even **parse** the new source (new syntax, not new behavior). The archived Rust compiler is no longer a fallback: it cannot lex the current source (it rejects the REPL's `"\e["` escape with `unterminated string`) and predates most of the language.

## Steps

1. Add lexer/parser/lowering support for the new form; do **not** use it anywhere in `stdlib/compiler/`, `selfhost/`, `stdlib/repl/`, `stdlib/lsp/builtins.zyl` or anything else `selfhost/driver.zyl` reaches through `use`.
2. `./boot.sh --bootstrap-from-self && ./boot.sh`; commit the seed.
3. Now use the syntax in compiler source; reseed again.

## Notes

- The same applies to new built-ins the compiler itself wants to call and to new escapes in string literals (the bitwise operators are still not constant-folded because the seed of the day could not compile a `bit-and` in the optimizer).

## See Also

- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
