# syn-naming-conventions

> kebab-case functions and variables, PascalCase types and variants, `?` predicates, `_` for unused, two-space indent; treat special-form names as reserved.

## Why It Matters

The standard library and compiler follow these conventions throughout, and the checker only half-enforces reserved names: `(let match 3 ...)` compiles, and some names (`when`, `unless`) are core library *functions*. Shadowing them makes code unreadable and can change which definition a call reaches.

## Conventions

| Category | Convention | Example |
|---|---|---|
| Functions, variables, params | kebab-case | `read-file`, `user-count` |
| Types, ADT variants | PascalCase | `Shape`, `Circle`, `Some` |
| Constants | nullary `defn`, kebab-case | `(defn max-size () 1000)` |
| Predicates | `?` suffix | `even?`, `origin?` |
| Mutating helpers/macros | `!` suffix | `swap!` |
| Unused names | `_` or `_`-prefix | `_`, `_rest` |
| Compiler internals | phase prefix | `ic-`, `cg-`, `mr-`, `sb-` |

Indentation is two spaces; the LSP formatter re-indents by paren depth.

## Bad

```lisp
(defn ReadFile (FileName) ...)   ; wrong case
(let begin 5 begin)              ; compiles, unreadable
(defn d1 (x d2) x)               ; dummy names: use _
```

## Good

```lisp
(defn read-file ((file-name String)) ...)
(defn first-of (a _) a)
```

## Notes

- `E_RESERVED_KEYWORD` is raised today only for the unimplemented 16/32/64-bit byte loads/stores (`load-u16`, `store-u32`, ...). The spec's full reserved list is not enforced.
- Names in the core prelude (`identity`, `const`, `flip`, `compose`, `apply`, `abs`, `max`, `min`, `clamp`, `signum`, `square`, `cube`, `xor`, `nand`, `nor`, `implies`, `when`, `unless`, `is-zero`, `is-even`, `is-odd`, `print-int`, `print-float`, `print-string`, `print-bool`, the `option-*`/`result-*`/`list-*` helpers, `car`/`cdr`...) are taken: redefining one is `E_DUPLICATE_DEFINITION`.

## See Also

- [data-no-redeclare-prelude](data-no-redeclare-prelude.md) - names you cannot reuse
- [fn-underscore-discard](fn-underscore-discard.md) - `_` rules
