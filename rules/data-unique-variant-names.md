# data-unique-variant-names

> Give every variant a name unique across all types in the program, and declare each type name exactly once.

## Why It Matters

Two `deftype`s may share a variant name without an error, but then: construction builds the variant of the **later** declaration, and the exhaustiveness checker **skips** every `match` whose arms use the shared name (it infers the scrutinee's type from constructor names and refuses to guess). You lose exhaustiveness checking silently. Duplicate type definitions produce incompatible constructor identities and matches against them fail silently; a second `deftype` with an existing name is `E_DUPLICATE_DEFINITION`.

## Bad

```lisp
(deftype Shape (Circle Int) (Square Int))
(deftype Token (Square) (Round))     ; Square shared: matches on it are unchecked
```

## Good

```lisp
(deftype Shape (Circle Int) (Square Int))
(deftype Token (TkSquare) (TkRound)) ; prefix variants, as the compiler does (Tk*, A*, I*)
```

## Notes

- The compiler's own naming conventions exist for this reason: tokens `Tk*`, AST `A*`/`Ast*`, ICNF `I*`, expression tree `E*`.
- A missing `(use ...)` can make a constructor "unknown" inside a module, which turns its arm into a catch-all: see [pkg-use-what-you-construct](pkg-use-what-you-construct.md).

## See Also

- [match-misspelled-last-arm](match-misspelled-last-arm.md) - unknown names become catch-alls
- [boot-one-deftype-per-name](boot-one-deftype-per-name.md)
