# syn-brackets-and-balance

> `()` and `{}` read as plain lists, `[a b]` reads as the list literal `(list a b)`; keep every opener matched, start every top-level form in column 1 and no nested form there, and check with `zyl balance`.

## Why It Matters

Before parsing, `sexp_balance.zyl` rejects unclosed openers (`E_UNBALANCED_UNCLOSED`), stray closers (`E_UNBALANCED_UNEXPECTED_CLOSE`) and mismatched kinds (`E_UNBALANCED_MISMATCHED_BRACKET`) with line, column and a fix-it hint. Since 2026-09-28 the check also enforces the layout rule of spec §1.6: a top-level form starts in column 1 and no nested opener does. A **misplaced** paren that leaves the file net-balanced, a missing closer balanced by an extra one later, is therefore caught too: the next top-level form starts in column 1 while the broken one is open, which is `E_UNBALANCED_UNCLOSED` at the broken form, with a fix-it naming where the indentation first goes wrong. A misplaced paren inside one form that keeps the layout intact is caught later where it can be: a `defn` whose parameter list swallows its body is `E_MALFORMED_PARAMETER` or `E_MALFORMED_FORM`. Run `zyl balance` after every edit ([tool-balance](tool-balance.md)).

## Bad

```lisp
(defn f (x (+ x 1)) x)     ; E_MALFORMED_PARAMETER: `(+ ...)` is not a parameter
(defn f2 (x (+ x 1)))      ; E_MALFORMED_FORM: malformed `defn` form

(defn g (x)
  (if (> x 0) x 0)         ; missing ) -- balanced by the extra ) below
(defn h () 1))             ; E_UNBALANCED_UNCLOSED at g: h starts in column 1 while g is open

(defn k (x)
(+ x 1))                   ; E_UNBALANCED_UNCLOSED: a nested form may not start in column 1
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
- A mismatched closer is reported at the closer with a label at the opener it failed to close; an unterminated string at its opening quote.
- Every file is balance-checked on its own; see [boot-parens-per-file](boot-parens-per-file.md).

## See Also

- [syn-list-literals-and-quote](syn-list-literals-and-quote.md) - `[...]` and quoted data
- [syn-no-stray-characters](syn-no-stray-characters.md) - stray bytes are a located `E_INVALID_CHAR`
- [boot-parens-per-file](boot-parens-per-file.md) - a missing closer swallows the rest of its file
