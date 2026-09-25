# pkg-native-dependencies

> Ship C code declaratively with `(native (sources ...) (cflags ...) (include-dirs ...) (link-libs ...))`; build scripts are forbidden.

## Why It Matters

Arbitrary build-time code would break determinism (§27), so native builds are declarative and flag-limited. Only the root package's native block is built today.

## Good

```lisp
(package
  (name "acme/fast") (version "0.1.0") (zyl "5.0") (edition "2026")
  (capabilities ffi native)
  (native (sources "c/fast.c") (cflags "-O2" "-DTRIPLE=3")))
```

```c
/* c/fast.c */
long long fast_triple(long long n) { return n * TRIPLE; }
```

```lisp
; fast.zyl
(extern "fast_triple" (Int) Int)       ; the C signature: without it the call is E_CANNOT_INFER
(defn triple ((n Int)) (ffi-call "fast_triple" n 1000))
(defn main () (begin (print (triple 14)) 0))
```

`zyl build && ./fast` prints 42 (verified 2026-09-25).

## Notes

- Flag allowlist and errors: see [ffi-linking](ffi-linking.md).
- `cc` runs with a canonical sorted argument vector; objects in `build/native/`. Each object is recorded in `<name>.buildinfo` `(native-objects ("build/native/x.c.o" "blake3:..."))` (package-relative path, so builds are directory-independent) and its hash feeds the final build hash ([pkg-mvs-lock-store](pkg-mvs-lock-store.md)).

## See Also

- [ffi-linking](ffi-linking.md)
