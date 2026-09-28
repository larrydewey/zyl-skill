# pkg-capabilities

> Declare the narrowest `(capabilities ...)` set a package needs; every definition is checked, `main` and top-level tests included, but a manifest-less file is not.

## Why It Matters

Absent `(capabilities ...)` means **none**. Using a gated construct or stdlib module without the capability is `E_PKG_CAPABILITY_VIOLATION`, naming both sides of the boundary. A declared set is a **ceiling on the declaring package**, not a grant along an edge: a caller without `ffi` may call a dependency's function that calls C.

| Capability | Grants |
|---|---|
| `io` | `file-open`, `file-read`, `file-write`, `file-close`, `read-line`; `core/io`, `stdlib/io` |
| `ffi` | `ffi-call`, `ffi-pin`, `ffi-unpin`; `stdlib/ffi` |
| `actor` | `spawn`, `chan`, `chan-send`, `chan-recv`; `stdlib/actor` |
| `secret` | the `Secret` type in package code; `stdlib/math/secret` |
| `native` | shipping/compiling C sources |
| `unsafe` | `:unsafe` imports (not enforced) |

`print` is not gated. The stdlib holds every capability and is never checked.

## Good

```lisp
(capabilities io ffi)          ; may do file IO and call C; may not spawn
(capabilities)                 ; pure computation
(deny-capabilities ffi native unsafe)   ; root: forbid graph-wide
```

## Limits

- The root package's `main` (which stays unqualified) is checked under a `<main>` grant, and top-level `test` forms are checked too (since 2026-09-28).
- A lone file without `zyl.pkg` declares nothing and is not checked.
- `zyl audit` lists each package's capabilities and the locked closure; `zyl update` reports closure growth.

## See Also

- [pkg-mvs-lock-store](pkg-mvs-lock-store.md)
- [ffi-linking](ffi-linking.md)
