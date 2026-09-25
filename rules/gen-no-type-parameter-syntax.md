# gen-no-type-parameter-syntax

> Never write the spec's type-parameter groups `((T) x)` or `((T : Ord) a b)`; leave parameters unannotated, or annotate them with an uppercase type variable such as `(a T)`.

## Why It Matters

Spec §6.1 generic syntax is not implemented, and both spellings are rejected:

| Written | Result |
|---|---|
| `((T) x)` | `E_MALFORMED_PARAMETER: `(T ...)` is not a parameter - write a name, or (name Type)` |
| `((T : Ord) a b)` | `E_MALFORMED_PARAMETER: a parameter's type is written (name Type), without a colon` |
| `((a Ord))` | `E_MALFORMED_PARAMETER: `Ord` is a trait, not a type` |

Trait bounds cannot be declared. They are not needed: a function that compares or calls a trait method on a type parameter is specialized per argument type, and a type with no impl is reported at the call (`E_TRAIT_NOT_FOUND`) ([gen-per-type-instances](gen-per-type-instances.md)).

## Bad

```lisp
(defn ident ((T) x) x)
(defn smallest ((T : Ord) a b) (if (< a b) a b))
```

## Good

```lisp
(defn first-of (a _) a)                ; polymorphic: accepts any types
(defn smaller (a b) (if (< a b) a b))  ; Int, Float or String
(defn pick ((c Bool) (a T) (b T)) (if c a b))  ; T: a and b have one type
```

## Notes

- `identity`, `min`, `max`, `compose` already exist in the prelude; reusing those names is `E_DUPLICATE_DEFINITION` ([data-no-redeclare-prelude](data-no-redeclare-prelude.md)).
- Book chapter 7 shows the spec syntax only to say it is rejected.

## See Also

- [gen-unannotated-is-polymorphic](gen-unannotated-is-polymorphic.md)
- [syn-keywords-and-symbols](syn-keywords-and-symbols.md)
