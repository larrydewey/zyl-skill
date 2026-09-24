# trait-coherence-and-orphans

> Declare the trait (`(trait Name ...)`) in your package before implementing it for a type you don't own, write each `(Trait, Type)` impl exactly once, and use `(impl-not Trait Target)` to forbid a pair.

## Why It Matters

The `trait` declaration types calls of its methods (impls are not checked against it), and the **orphan rule** is enforced at the package boundary: an impl is legal only if the package defines the trait or the type. `(impl Describe Int ...)` without a local `(trait Describe ...)` is `E_PKG_ORPHAN_IMPL` (neither `Int` nor an undeclared trait is yours). Coherence C1 (one impl per pair) is not checked by a pass: a duplicate impl fails **in the assembler** ("symbol ... is already defined"), not with `E_DUPLICATE_IMPL`. A call with no impl at all fails at **link time**, not `E_TRAIT_NOT_FOUND`.

`(impl-not Trait Target)` is a top-level declaration checked over the whole program after module resolution. `Target` is a type, or a trait (then every implementor, in any declaration order). Any `impl` or `derive` of the forbidden pair, in any module or package, is `E_IMPL_FORBIDDEN`. Its **flow rule** closes the wrapper loophole: the result of an impl of `Trait` for *any* type may not derive from a protected value (a value of the target type, a field of that type, a `match` binder of one) — also `E_IMPL_FORBIDDEN`. For `Show`, protected types get a compiler-made `Show` printing `<hidden>`, and a derived `Show` prints such fields as `<hidden>`. The prelude declares `(impl-not Show Secret)`, so a `Show` impl/derive for a Secret type is forbidden (it prints `<secret>`). `declassify` is the explicit escape from the flow rule.

## Bad

```lisp
(impl Describe Int (defn describe (self) self))
;; E_PKG_ORPHAN_IMPL: impl of Describe for Int where neither the trait nor the type is local
```

```lisp
(defstruct Handle (fd Int))
(trait Dump (dump (self) Int))
(impl-not Dump Handle)
(impl Dump Handle (defn dump (self) self.fd))
;; error[E_IMPL_FORBIDDEN]: `Dump` cannot be implemented for `Handle`
(defstruct Conn (host String) (h Handle))
(impl Dump Conn (defn dump (self) self.h.fd))
;; PANIC: E_IMPL_FORBIDDEN: the result of `Dump.dump for Conn` is derived from a value that an (impl-not Dump ...) declaration protects
```

## Good

```lisp
(trait Describe (describe self))
(impl Describe Int (defn describe (self) self))

(impl Dump Conn (defn dump (self) (str-len self.host)))   ; public fields only: allowed
```

## Notes

- Within one package, any module may implement any of the package's traits for any of its types.
- The direct `E_IMPL_FORBIDDEN` is a located `error[...]`; the flow-rule violation is an unlocated `PANIC:` line naming the impl.
- An impl for a generic type names the bare type (`(impl Show Vec ...)`) and covers every instantiation, so C3 overlap cannot arise except as a C1 duplicate.

## See Also

- [pkg-capabilities](pkg-capabilities.md)
- [trait-static-dispatch](trait-static-dispatch.md)
