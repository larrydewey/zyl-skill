# pkg-manifest

> Write `zyl.pkg` with `name`, `version`, `zyl` and `edition`, scoped names, strict SemVer, and **bare minimum** version requirements.

## Why It Matters

The manifest is an S-expression read by the language's own parser. A missing required field is `E_MANIFEST_INVALID`; a range operator (`^`, `~`, `>=`) is `E_PKG_BAD_REQUIREMENT` because MVS takes minimums.

## Good

```lisp
(package
  (name "acme/json") (version "1.4.0")
  (zyl "5.0") (edition "2026")
  (description "...") (license "MIT") (repository "...")
  (capabilities io)
  (deps
    (dep "core/bytes" "2.1.0")
    (dep "acme/utf8"  "1.0.0" (features utf16))
    (dep "acme/dev"   "0.3.0" (path "../dev"))
    (dep "acme/git"   "1.0.0" (git "https://..." (rev "a1b2c3d"))))
  (dev-deps (dep "acme/quickcheck" "2.0.0"))
  (features (feature utf16 (deps (dep "acme/utf16" "1.0.0"))) (feature simd))
  (native (sources "c/fastpath.c") (cflags "-O2") (link-libs "m")))
```

## Rules

- Names: two segments `owner/name` (`E_PKG_BAD_NAME`); major ≥ 2 adds a third, `acme/json/v2`, making majors distinct packages.
- Versions: `MAJOR.MINOR.PATCH[-pre]` (`E_PKG_BAD_VERSION`).
- Two `dep` entries for one name: `E_PKG_DUPLICATE_DEP`.
- `(zyl "5.0")` is a minimum compiler (`E_PKG_COMPILER_TOO_OLD`); the only edition is `"2026"` (`E_PKG_UNKNOWN_EDITION`).
- Root-only fields: `(overrides ...)`, `(deny-capabilities ...)`.
- Without a `zyl.pkg`, a file is package `local/main` and nothing is enforced.
- `zyl new owner/name` creates `name/zyl.pkg` and `name/name.zyl`.

## See Also

- [pkg-mvs-lock-store](pkg-mvs-lock-store.md)
- [pkg-capabilities](pkg-capabilities.md)
