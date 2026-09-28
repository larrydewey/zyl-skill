# syn-naming-conventions

> kebab-case functions and variables, PascalCase types and variants, `?` predicates, `_` for unused, two-space indent; treat special-form names as reserved.

## Why It Matters

The standard library and compiler follow these conventions throughout, and the checker enforces the reserved keywords (`(let match 3 ...)` is `E_RESERVED_KEYWORD`), but some names (`when`, `unless`) are core library *functions* and can still be shadowed. Shadowing them makes code unreadable and can change which definition a call reaches.

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
(let begin 5 begin)              ; E_RESERVED_KEYWORD
(defn d1 (x d2) x)               ; dummy names: use _
```

## Good

```lisp
(defn read-file ((file-name String)) ...)
(defn first-of (a _) a)
```

## Notes

- The spec 1.3 keywords are reserved (`E_RESERVED_KEYWORD`, since 2026-09-28): none may name a function, parameter, `let` binding, type, variant, field, trait, alias, macro or module, so `(let match 3 ...)` and `(defn setup () ...)` are errors. `make-S` is reserved when the file declares a struct `S`. The standard library and the runtime are exempt.
- `unit` is the Unit value and cannot be rebound: `(let unit 3 (print unit))` compiles, warns `W_UNUSED_VARIABLE`, and prints the Unit value (`0`), not 3.
- `list`, `quote` and `quasiquote` head the reader's list forms: a variable named `list` works, but `(list ...)` is always the list literal.
- A program type may not reuse a prelude constructor name (`Some`, `None`, `Ok`, `Err`, `Cons`, `Nil`): `E_DUPLICATE_VARIANT`.
- Names in the core prelude (`identity`, `const`, `flip`, `compose`, `apply`, `abs`, `max`, `min`, `clamp`, `signum`, `square`, `cube`, `xor`, `nand`, `nor`, `implies`, `when`, `unless`, `is-zero`, `is-even`, `is-odd`, `print-int`, `print-float`, `print-string`, `print-bool`, the `option-*`/`result-*`/`list-*` helpers, `car`/`cdr`...) are taken: redefining one is `E_DUPLICATE_DEFINITION`.

## See Also

- [data-no-redeclare-prelude](data-no-redeclare-prelude.md) - names you cannot reuse
- [fn-underscore-discard](fn-underscore-discard.md) - `_` rules
