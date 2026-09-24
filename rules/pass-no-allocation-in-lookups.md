# pass-no-allocation-in-lookups

> Never allocate inside a comparison or lookup that runs per element of a table: the arena never frees, so every temporary string is kept for the whole compile.

## Why It Matters

Compiler tables are association lists searched linearly, and type inference searches them for nearly every node. A comparison that builds a string (`str-concat`, `str-substring`) allocates once per element per lookup, and nothing is ever reclaimed ([own-heap-never-freed](own-heap-never-freed.md)). On 2026-09-24 one such helper took a self-compile from about 0.6 GB to 2 GB of RSS.

## Bad

```lisp
; two allocations per comparison
(defn type-name-matches (n name)
  (or (> (str-eq n name) 0)
      (has-suffix n (str-concat "::" name))))   ; has-suffix calls str-substring
```

## Good

```lisp
; compare in place: a runtime helper, or byte-at loops over the originals
(defn type-name-matches (n name)
  (> (ffi-call "zyl_cstr_key_matches" n name 1000) 0))
```

## Notes

- The same holds for building output: padding or text built one character at a time with `str-concat` is quadratic in bytes. Build by halving (`space-run`) or in C.
- `./boot.sh` caps each stage at 2 GB (`ZYL_STAGE_MEMORY`); `E_OUT_OF_MEMORY` there is this bug until proven otherwise. Find it by stage (`ZYL_DEBUG_STAGES=1` while watching RSS), then by reverting files.
- A new runtime helper must also be added to the interpreter's FFI table (the `X(...)` list in `actor_runtime.c`) so `zyl eval` and the REPL can call it, and to `in-ffi-returns-str` in `stdlib/repl/interp.zyl` if it returns a string.

## See Also

- [pass-avoid-repeated-subtree-work](pass-avoid-repeated-subtree-work.md)
- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
