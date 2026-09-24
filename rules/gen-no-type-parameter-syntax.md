# gen-no-type-parameter-syntax

> Never write the spec's type-parameter groups `((T) x)` or `((T : Ord) a b)`; leave parameters unannotated instead.

## Why It Matters

Spec §6.1 generic syntax is not implemented, and both spellings fail in different ways:

| Written | Result |
|---|---|
| `((T) x)` | `E_MALFORMED_PARAMETER`: `(T ...)` is not a parameter |
| `((T : Ord) a b)` | the lexer merges `: Ord` into keyword `:Ord`, making `(T :Ord)` an ordinary **value** parameter named `T`; the function takes one more argument, and `(smallest 3 5)` is `E_ARITY_MISMATCH` |

Bounds therefore cannot be declared or checked at all.

## Bad

```lisp
(defn identity ((T) x) x)
(defn smallest ((T : Ord) a b) (if (< a b) a b))
```

## Good

```lisp
(defn first-of (a _) a)              ; polymorphic: accepts any types
(defn smaller (a b) (if (< a b) a b))
```

## Notes

- `identity`, `min`, `max`, `compose` already exist in the prelude; reusing those names is `E_DUPLICATE_DEFINITION`.
- Appendix E of the book still shows the spec syntax in a Rust comparison; the chapters are authoritative: it does not compile.
- The monomorphizer treats a parameter whose name starts with an uppercase letter as a type parameter; avoid uppercase value-parameter names.

## See Also

- [gen-unannotated-is-polymorphic](gen-unannotated-is-polymorphic.md)
- [syn-keywords-and-symbols](syn-keywords-and-symbols.md)
