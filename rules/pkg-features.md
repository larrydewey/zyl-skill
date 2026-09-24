# pkg-features

> Use features only to **add** top-level definitions or impls via top-level `(feature-gate f def)`; never to replace one.

## Why It Matters

Features are unified across the graph: the union of all requests is computed and the package compiled once with it (recorded in the lock). A gated definition that collides with a base definition is `E_PKG_FEATURE_COLLISION`; requesting an undeclared feature is `E_PKG_FEATURE_UNKNOWN`. A nested `feature-gate` is not rejected but never reaches the resolver's top-level scan and is compiled as an ordinary form.

## Good

```lisp
; acme/feat -- feat.zyl
(pub defn base (n) (+ n 1))
(feature-gate simd (pub defn fast (n) (* n 8)))

; manifest
(features (feature simd) (feature utf16))

; dependent
(deps (dep "acme/feat" "1.0.0" (features simd) (path "../lib")))
```

## Notes

- An optional dependency named in a `(feature ...)` enters the graph only when that feature is in the union.

## See Also

- [pkg-manifest](pkg-manifest.md)
