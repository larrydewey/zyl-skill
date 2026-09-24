# pass-string-eq-in-compiler

> In compiler source, compare dynamically built strings (names, keys, type strings) with `str-eq`, not `=`.

## Why It Matters

`=` on strings compares contents only when codegen knows both operands are strings; on dynamically built strings in untyped positions it compares **pointers**. In the compiler this has two failure modes: lookups silently miss (some type-inference name lookups do exactly this, which is why REPL `:type` reports builtin applications as *unresolved*), and results depend on allocation order, which differs between stage 2 and stage 3 — historically a fixed-point breaker, fixed by `zyl_cstr_eq` for String-kind operands.

## Notes

- Fixing the inference lookups changes control flow deep in generic-instantiation tracking; see the header of `stdlib/lsp/compiler_bridge.zyl` before attempting it.

## See Also

- [fn-string-equality](fn-string-equality.md)
- [det-no-address-dependent-output](det-no-address-dependent-output.md)
