# ffi-linking

> Link your own C either with a package `native` block (preferred) or by emitting assembly and running `cc` yourself; the single-file CLI takes no extra objects or libraries.

## Why It Matters

`zyl file.zyl` always links with exactly `cc -no-pie prog.s actor_runtime.c -o prog -lpthread`. The CLI accepts only `-o` and `--emit-asm`; any other word after the source becomes the output path. A missing C symbol is an ordinary linker error.

## Good: by hand

```bash
zyl ffi-demo.zyl --emit-asm -o ffi-demo.s
cc -no-pie ffi-demo.s mylib.c ~/.zyl/actor_runtime.c -o ffi-demo -lpthread -lm
```

`actor_runtime.c` is in `~/.zyl` after install, or `build/boot/` / `runtime/` in a checkout. `-no-pie` is required.

## Good: package native block

```lisp
(package
  (name "demo/mathlib") (version "0.1.0") (zyl "5.0") (edition "2026")
  (capabilities ffi native)
  (native (sources "c/mathlib.c") (cflags "-O2" "-std=c11") (link-libs "m")))
```

`zyl build` compiles sources with `cc -c` into `build/native/` and links them.

## Rules for native blocks

- Both capabilities: `native` to ship sources, `ffi` to call them (`E_PKG_CAPABILITY_VIOLATION` otherwise).
- `cflags` allowlist: `-O*`, `-D*`, `-std=*`, `-fPIC`, `-fno-strict-aliasing`, `-fwrapv`, `-fstack-protector-strong`, `-fno-omit-frame-pointer`. Anything else (`-I`, `-L`, `-l`, `-Wl,`) is `E_PKG_NATIVE_FLAG_DENIED`; use `(include-dirs ...)` and `(link-libs ...)`.
- Paths are package-relative (`E_PKG_NATIVE_PATH_ESCAPE`); `cc` failure is `E_PKG_NATIVE_BUILD_FAILED`.
- Only the **root** package's native block is built today; dependencies' native sources are not linked.
- No build scripts, ever.

## See Also

- [pkg-native-dependencies](pkg-native-dependencies.md)
- [tool-cli-arguments](tool-cli-arguments.md)
