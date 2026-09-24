# pkg-canonical-keys

> Read linker symbols and resolver messages as canonical keys `<package>@<major>::<module>::<symbol>`, mangled injectively.

## Why It Matters

Every top-level definition is qualified to a canonical key, so two packages (or two modules) can define the same name. Diagnostics (`in local/main@0::app::check`) and assembly labels use these keys.

```
acme/greet@0::greet::greet
zyl/std@5::collections/vec::vec-push
local/main@0::hello::main-helper          ; a lone file is package local/main
```

Mangling keeps letters/digits, `_` → `_5F`, other bytes → `_xHH`:

```
acme/greet@0::greet::greet-twice  ->  zy_acme_x2Fgreet_0__greet__greet_x2Dtwice
```

Keys over 200 bytes become a 184-byte prefix plus 16 hex digits of BLAKE3.

## Notes

- Never qualified: `main` (label `_ZYL_main`) and the inlined string built-ins `str-concat`, `str-length`, `str-substring`, `str-equal`.
- The key depends on the source file's base name (its module path), so assembly depends on the file name but not its directory or the `-o` path.

## See Also

- [cg-symbols-and-entry](cg-symbols-and-entry.md)
