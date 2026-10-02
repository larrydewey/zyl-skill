# icnf-lowering-map

> Know what each source form lowers to, so you can predict codegen and write passes over ICNF.

| Source | ICNF |
|---|---|
| integer, `true`/`false`, byte literal | `IConst` |
| string / float literal | `IStr` / `IFlt` (source text) |
| variable | `ILoad` (a name the type pass renamed, e.g. to an instance `f~T`, loads the new name) |
| `let`, `let-mut` | `ILet` (mutability gone; `with-resource`, `assert-fail`, `let*`, `vec`, `map`, `div?`/`rem?`, `test-suite` and `test-property` were rewritten on the parse tree, `compiler/desugar.zyl`) |
| `set!` | `ISet` |
| Int `+ - *` | `IBinop` 0-2 (checked: a `jo` to the overflow trap); under the file's `(numeric wrapping)` or `(numeric saturating)`, `ic-policy-op` turns them into 18-20 or 21-23. Float operands keep 0-2 (IEEE, outside the policy). N-ary forms fold left (`ic-binop-fold`) |
| `wrapping+ - *`, `saturating+ - *` | `IBinop` 18-20, 21-23, whatever the policy |
| `/`, `%`, `div!`, `rem!` | `IBinop` 3, 4 (`ic-op-of` maps `div!` to 3 and `rem!` to 4; the numeric check has already required `/` and `%` to have a nonzero literal divisor) |
| `div?`, `rem?` | rewritten on the parse tree (`ds-div-opt`): `(let x a (let y b (if (= y 0) None (Some (div! x y)))))` |
| `begin` / `if` / `while` | `ISeq` / `IIf` / `IWhile` |
| `for` | nested `ILet`s around `IWhile` |
| `(list a b)`, `[a b]`, `'(1 2)`, quasiquote | never reach lowering as such: qualification (`qf-special`) and `convert-ast` rewrite them to `Cons`/`Nil` constructions (`ast-list-literal`, `ast-quote-data`); a `,@` splice becomes a `zyl-qq-append` call |
| known function call | `ICall` (to the instance name when the type pass specialized it, `node-calls`) |
| call through a local / computed head | `ICall` on the local; computed head bound to `_callee_N` first |
| `ffi-call` | after `ffi-check-call`: `zyl_*` symbol: `IFfi sym args` (timeout dropped); a call written without a timeout already carries its `extern`'s `:timeout N`, filled in by `convert-ast` (`ffi-default-timeout`, `expr_inner.zyl`); other symbol: `IFfi "zyl_ffi_timed" (ISymAddr sym, IStr sym, IConst ms, IConst argc, args...)` |
| `with-region` | `parse-with-region` (Expr level, `E_REGION_SPEC` on a bad spec) then `IRegion kind block align limit body` (kind 1 arena, 2 fixed) via `ic-with-region` |
| string built-ins, file I/O, byte buffers, atomics, `spawn`, the channel forms, `ffi-pin` | `IFfi` to runtime functions (`zyl_cstr_concat`, `zyl_file_open_c`, `zyl_bytebuf_new`, `zyl_actor_spawn`, `zyl_chan_send`...); `chan-send` reuses the `ESend` node |
| bit intrinsics (`bit-popcount` ...) | `IFfi` to a `%` primitive the backends expand inline (`rt_mode.zyl`, `intrinsic-prim`); the interpreter evaluates a portable reference |
| constructor / struct construction | `IVariant name tag fields` (a nullary constructor too: a one-word block) |
| `match` | `IMatch` of `IArm`s (wildcard tag -1); nested patterns are rejected earlier (`E_NESTED_PATTERN`) |
| `struct-get` | `IMatch` with one arm per struct type having that field |
| `(Point.x p)` | an ordinary `ICall`: `convert-ast` emits a typed accessor `defn` for each field after the `defstruct` (`struct-accessor-forms`, `expr_inner.zyl`) |
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
