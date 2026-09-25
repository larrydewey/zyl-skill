# pkg-pub-and-explicit-imports

> Mark the public surface with inline `(pub defn ...)` and import with explicit `{ ... }` lists.

## Why It Matters

Visibility has two levels: package-private (default, visible to every module of the package) and `pub` (visible to dependents). `pub` prefixes a definition inline; it is not a wrapper list. `(export sym)` is deprecated and **dropped without effect**: exporting a non-`pub` definition still fails with `E_PKG_PRIVATE_SYMBOL`. Explicit import lists are checked at resolution time; with `*` or a bare path, a private symbol is not diagnosed by the resolver; the type checker then reports the call as a located `E_UNBOUND_VARIABLE` (`call to undefined function `helper``), which does not say the function exists but is private.

## Good

```lisp
; greet/greet.zyl -- root module of acme/greet
(pub defn greet (n) (+ n 100))
(defn helper (n) (* n 2))                    ; package-private
(pub defn greet-twice (n) (helper (greet n)))

; hello/hello.zyl
(use acme/greet { greet => hi greet-twice })
```

## Notes

- `pub` applies to `defn`, `def`, `deftype` (with all constructors), `defstruct`, `defstruct+`, `trait`, `defmacro`.
- Every definition, public or private, has its own canonical key, so renaming a private definition cannot break consumers.

## See Also

- [pkg-modules-and-use](pkg-modules-and-use.md)
- [pkg-canonical-keys](pkg-canonical-keys.md)
