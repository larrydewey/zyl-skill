# data-unique-variant-names

> Give every variant a name unique across all types in the program, and declare each type name exactly once.

## Why It Matters

Two `deftype`s may share a variant name without an error, but the name then means only the variant of the **later** declaration, in construction and in patterns alike. The earlier type's variant becomes unreachable: you cannot build it, a `match` over the earlier type cannot name it (`E_TYPE_MISMATCH: cannot unify Shape with Token`), and the exhaustiveness checker still demands it (`E_NON_EXHAUSTIVE_MATCH: match over `Shape` does not cover variant `Square``), so only a `_` arm can cover it. A second `deftype` with an existing type name is `E_DUPLICATE_DEFINITION`; a prelude constructor name (`Some`, `Cons`, ...) is `E_DUPLICATE_VARIANT`.

## Bad

```lisp
(deftype Shape (Circle Int) (Square Int))
(deftype Token (Square) (Round))     ; Square now means Token's Square only

(defn side (s) (match s (Circle n n) (Square n n)))
;; E_TYPE_MISMATCH: cannot unify Shape with Token
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
