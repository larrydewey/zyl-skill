# icnf-lowering-map

> Know what each source form lowers to, so you can predict codegen and write passes over ICNF.

| Source | ICNF |
|---|---|
| integer, `true`/`false`, byte literal | `IConst` |
| string / float literal | `IStr` / `IFlt` (source text) |
| variable | `ILoad` (a name the type pass renamed, e.g. to an instance `f~T`, loads the new name) |
| `let`, `let-mut`, `with-resource` | `ILet` (mutability gone) |
| `set!` | `ISet` |
| `begin` / `if` / `while` | `ISeq` / `IIf` / `IWhile` |
| `for` | nested `ILet`s around `IWhile` |
| `(list a b)`, `[a b]`, `'(1 2)`, quasiquote | never reach lowering as such: qualification (`qf-special`) and `convert-ast` rewrite them to `Cons`/`Nil` constructions (`ast-list-literal`, `ast-quote-data`); a `,@` splice becomes a `zyl-qq-append` call |
| known function call | `ICall` (to the instance name when the type pass specialized it, `node-calls`) |
| call through a local / computed head | `ICall` on the local; computed head bound to `_callee_N` first |
| `ffi-call` | after `ffi-check-call`: `zyl_*` symbol: `IFfi sym args` (timeout dropped); other symbol: `IFfi "zyl_ffi_timed" (ISymAddr sym, IStr sym, IConst ms, IConst argc, args...)` |
| `with-region` | `parse-with-region` (Expr level, `E_REGION_SPEC` on a bad spec) then `IRegion kind block align limit body` (kind 1 arena, 2 fixed) via `ic-with-region` |
| string built-ins, file I/O, byte buffers, atomics, `spawn`, `send`, `ffi-pin` | `IFfi` to runtime functions (`zyl_cstr_concat`, `zyl_file_open_c`, `zyl_bytebuf_new`, `zyl_actor_spawn`...) |
| constructor / struct construction | `IVariant name tag fields` (a nullary constructor too: a one-word block) |
| `match` | `IMatch` of `IArm`s (wildcard tag -1); nested patterns are rejected earlier (`E_NESTED_PATTERN`) |
| `struct-get` | `IMatch` with one arm per struct type having that field |
| `print` | `IPrint`, one per argument; a value with a `Show` impl prints its `Show.show` call (`node-shows`), marked String |
| `try`/`catch` | `ITryCatch` |
| `fn` | `IFn` lifted by `ic-hoist` to top level (name from `zyl_fresh_id`), referenced by `ILoad`; capturing: closure block |
| `assert-equal` | `IIf` of a call to the type's `==` instance when the type pass named one (else `IBinop 9`, or an approximate compare for float literals), `zyl_panic` on failure |
| `assert-true`/`-false`/`assert` | `IIf` calling `zyl_panic` on failure |
| top-level `(test "n" body)` | function `_test_<n>` + `zyl_register_test` in implicit `main` |
| a form with no lowering case | `(IConst 0)`, silently ([icnf-new-form-needs-case](icnf-new-form-needs-case.md)) |

## Tags

Variant tags come from `VTable` (`ast.zyl`): per-`deftype` 0-based in declaration order; structs from a global counter (100000+) so `struct-get` can dispatch by tag. A later `deftype` reusing a variant name shadows the earlier; the type checker now rejects the mixed uses that shadowing used to miscompile ([boot-one-deftype-per-name](boot-one-deftype-per-name.md)).

## Closures

`ic-lambda` lowers the body, reads free names off the lowered tree (`ic-lambda-free`: loads/call heads not params, not bound inside, not top-level functions). None: plain lifted function. Some: `[ic-closure-magic, code, env]`, env captured by value; the lifted function takes a trailing `_clos_env` and reads captures with `zyl_variant_field`.

## After lowering

The ICNF list then goes through inlining and copy propagation, folding, region inference and the reuse pass before codegen ([icnf-optimizer-scope](icnf-optimizer-scope.md)); the reuse pass may add owning clones `f~own` ([icnf-reuse-pass](icnf-reuse-pass.md)). There is no flag that prints the ICNF of a compile; `icnf_print.zyl` is used only by package builds (`zyl build` hashes the canonical text).

## See Also

- [icnf-tree-structure](icnf-tree-structure.md)
- [cg-closure-call-protocol](cg-closure-call-protocol.md)
