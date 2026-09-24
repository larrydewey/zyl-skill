# syn-brackets-and-balance

> `()`, `[]` and `{}` all read as the same list; keep every opener matched, and know the balance check only catches net imbalance.

## Why It Matters

Before parsing, `sexp_balance.zyl` rejects unclosed openers (`E_UNBALANCED_UNCLOSED`), stray closers (`E_UNBALANCED_UNEXPECTED_CLOSE`) and mismatched kinds (`E_UNBALANCED_MISMATCHED_BRACKET`) with line, column and a fix-it hint. A **misplaced** paren that leaves the file net-balanced is not a balance error. The common case, a `defn` whose parameter list swallows its body, is caught as `E_MALFORMED_PARAMETER`. Other shapes silently re-nest forms: a missing closer nests every following `defn` inside the broken one, and they vanish from the output.

## Bad

```lisp
(defn f (x (+ x 1))        ; E_MALFORMED_PARAMETER: body inside the param list

(defn g (x)
  (if (> x 0) x 0)         ; missing ) -- if balanced later, h is nested inside g
(defn h () 1))
```

## Good

```lisp
(defn f (x) (+ x 1))
(defn g (x)
  (if (> x 0) x 0))
(defn h () 1)
```

## Notes

- Brackets have no special meaning; the containing form decides: `{ a b }` is an import list, `[Eq Ord]` a trait-name list. There are no vector or map literals.
- Symptom of a silent re-nest: a function "missing from compiled output", or `E_UNBOUND_VARIABLE` for a function you can see in the file.
- For compiler source, also check each file on its own: see [boot-parens-per-file](boot-parens-per-file.md).

## See Also

- [syn-no-stray-characters](syn-no-stray-characters.md) - the other silent truncation
- [boot-parens-per-file](boot-parens-per-file.md) - per-file balance in the bundle
