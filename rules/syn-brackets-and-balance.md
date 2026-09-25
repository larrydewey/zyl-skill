# syn-brackets-and-balance

> `()` and `{}` read as plain lists, `[a b]` reads as the list literal `(list a b)`; keep every opener matched, and know the balance check only catches net imbalance.

## Why It Matters

Before parsing, `sexp_balance.zyl` rejects unclosed openers (`E_UNBALANCED_UNCLOSED`), stray closers (`E_UNBALANCED_UNEXPECTED_CLOSE`) and mismatched kinds (`E_UNBALANCED_MISMATCHED_BRACKET`) with line, column and a fix-it hint. A **misplaced** paren that leaves the file net-balanced is not a balance error. The common cases are caught later: a `defn` whose parameter list swallows its body is `E_MALFORMED_PARAMETER` when a body form follows, and `E_MALFORMED_FORM` ("malformed `defn` form") when none does. Other shapes silently re-nest forms: a missing closer nests every following `defn` inside the broken one. A nested `defn` defines nothing and contributes a wrong value to the enclosing body.

## Bad

```lisp
(defn f (x (+ x 1)) x)     ; E_MALFORMED_PARAMETER: `(+ ...)` is not a parameter
(defn f2 (x (+ x 1)))      ; E_MALFORMED_FORM: malformed `defn` form

(defn g (x)
  (if (> x 0) x 0)         ; missing ) -- balanced by the extra ) below
(defn h () 1))             ; h is nested in g: (g 1) returns 0, and (h) is E_UNBOUND_VARIABLE
```

## Good

```lisp
(defn f (x) (+ x 1))
(defn g (x)
  (if (> x 0) x 0))
(defn h () 1)
```

## Notes

- `[` is no longer a neutral bracket. `[1 2 3]` is `(list 1 2 3)`, a `List` ([syn-list-literals-and-quote](syn-list-literals-and-quote.md)); `[e]` in expression position is a one-element list, not a grouping. A derive's `[Eq Ord]` still names traits (the reader's `list` head is dropped there).
- `{ ... }` still has no meaning of its own; the containing form decides: `(use m { a b })` is an import list. In expression position `{f 1}` is the call `(f 1)`, and `{1 2}` is a call of `1` (`E_TYPE_MISMATCH`). Do not use braces outside import lists.
- The balance checker also enforces kinds: `[1 2)` and `(+ 1 2]` are `E_UNBALANCED_MISMATCHED_BRACKET`, with the fix-it naming the expected closer.
- Symptom of a silent re-nest: a function "missing from compiled output", `E_UNBOUND_VARIABLE` for a function you can see in the file, or a function returning 0 where its last visible form has another value.
- Every file is balance-checked on its own; see [boot-parens-per-file](boot-parens-per-file.md).

## See Also

- [syn-list-literals-and-quote](syn-list-literals-and-quote.md) - `[...]` and quoted data
- [syn-no-stray-characters](syn-no-stray-characters.md) - stray bytes are a located `E_INVALID_CHAR`
- [boot-parens-per-file](boot-parens-per-file.md) - a missing closer swallows the rest of its file
