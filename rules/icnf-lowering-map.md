# icnf-lowering-map

> Know what each source form lowers to, so you can predict codegen and write passes over ICNF.

| Source | ICNF |
|---|---|
| integer, `true`/`false`, byte literal | `IConst` |
| string / float literal | `IStr` / `IFlt` (source text) |
| variable | `ILoad` |
| `let`, `let-mut`, `with-resource` | `ILet` (mutability gone) |
| `set!` | `ISet` |
| `begin` / `if` / `while` | `ISeq` / `IIf` / `IWhile` |
| `for` | nested `ILet`s around `IWhile` |
| known function call | `ICall` |
| call through a local / computed head | `ICall` on the local; computed head bound to `_callee_N` first |
| `ffi-call` | `IFfi` with the last argument dropped (timeout) |
| string built-ins, file I/O, byte buffers, atomics, `spawn`, `send`, `ffi-pin` | `IFfi` to runtime functions (`zyl_cstr_concat`, `zyl_file_open_c`, `zyl_bytebuf_new`, `zyl_actor_spawn`...) |
| constructor / struct construction | `IVariant name tag fields` |
| `match` | `IMatch` of `IArm`s (wildcard tag -1); nested field patterns get a fresh name and an inner `IMatch` |
| `struct-get` | `IMatch` with one arm per struct type having that field |
| `try`/`catch` | `ITryCatch` |
| `fn` | `IFn` lifted by `ic-hoist` to top level (name from `zyl_fresh_id`), referenced by `ILoad`; capturing → closure block |
| `assert-equal`/`-true`/`-false` | `IIf` calling `zyl_panic` on failure |
| top-level `(test "n" body)` | function `_test_<n>` + `zyl_register_test` in implicit `main` |
| unrecognized form | `(IConst 0)` |

## Tags

Variant tags come from `VTable` (`ast.zyl`): per-`deftype` 0-based in declaration order; structs from a global counter (100000+) so `struct-get` can dispatch without static types. A later `deftype` reusing a variant name shadows the earlier.

## Closures

`ic-lambda` lowers the body, reads free names off the lowered tree (`ic-lambda-free`: loads/call heads not params, not bound inside, not top-level functions). None → plain lifted function. Some → `[ic-closure-magic, code, env]`, env captured by value; lifted fn takes trailing `_clos_env` and reads captures with `zyl_variant_field`.

## See Also

- [icnf-tree-structure](icnf-tree-structure.md)
- [cg-closure-call-protocol](cg-closure-call-protocol.md)
