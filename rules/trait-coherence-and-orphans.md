# trait-coherence-and-orphans

> Declare the trait (`(trait Name ...)`) in your package before implementing it for a type you don't own, and write each `(Trait, Type)` impl exactly once.

## Why It Matters

The `trait` declaration types calls of its methods (impls are not checked against it), and the **orphan rule** is enforced at the package boundary: an impl is legal only if the package defines the trait or the type. `(impl Describe Int ...)` without a local `(trait Describe ...)` is `E_PKG_ORPHAN_IMPL` (neither `Int` nor an undeclared trait is yours). Coherence C1 (one impl per pair) is not checked by a pass: a duplicate impl fails **in the assembler** ("symbol ... is already defined"), not with `E_DUPLICATE_IMPL`. A call with no impl at all fails at **link time**, not `E_TRAIT_NOT_FOUND`.

## Bad

```lisp
(impl Describe Int (defn describe (self) self))
;; E_PKG_ORPHAN_IMPL: impl of Describe for Int where neither the trait nor the type is local
```

## Good

```lisp
(trait Describe (describe self))
(impl Describe Int (defn describe (self) self))
```

## Notes

- Within one package, any module may implement any of the package's traits for any of its types.
- An impl for a generic type names the bare type (`(impl Show Vec ...)`) and covers every instantiation, so C3 overlap cannot arise except as a C1 duplicate.

## See Also

- [pkg-capabilities](pkg-capabilities.md)
- [trait-static-dispatch](trait-static-dispatch.md)
