# pass-no-allocation-in-lookups

> Never allocate inside a comparison or lookup that runs per element of a table: in the compiler most such temporaries still land in the heap, which is never freed, so every one is kept for the whole compile.

## Why It Matters

Many compiler tables are association lists searched linearly, and the type pass searches such tables for nearly every node. A comparison that builds a string (`str-concat`, `str-substring`) allocates once per element per lookup. Per-call regions reclaim a temporary only when region inference proves it dies in the call; in the compiler, whose tables are global and whose data mostly escapes into results, that often fails, and heap memory is never reclaimed ([own-heap-never-freed](own-heap-never-freed.md)). On 2026-09-24 one such helper took a self-compile from about 0.6 GB to 2 GB of RSS; `zyl_cstr_key_matches` (the key equals the name, or ends in `::name`, compared in place) is the fix that stayed.

## Bad

```lisp
; two allocations per comparison
(defn type-name-matches (n name)
  (or (str-eq n name)
      (has-suffix n (str-concat "::" name))))   ; has-suffix calls str-substring
```

## Good

```lisp
; compare in place: a runtime helper, or byte-at loops over the originals
(defn type-name-matches (n name)
  (ffi-call "zyl_cstr_key_matches" n name 1000))   ; String String -> Bool
```

## Notes

- The same holds for building output: padding or text built one character at a time with `str-concat` is quadratic in bytes. Build by halving (`space-run`) or in C.
- `./boot.sh` caps each stage at 4 GB of cumulative allocation (`ZYL_STAGE_MEMORY`, passed on as `ZYL_MAX_MEMORY`); a self-compile allocates somewhat over 2 GB since inlining, reuse and the native backend landed, so `E_OUT_OF_MEMORY` there is this bug until proven otherwise. Find it by stage (`ZYL_DEBUG_STAGES=1` appends each phase name to `/tmp/dbg`; watch RSS alongside), then by reverting files.
- Fixpoint passes are the other place this bites: the reuse pass keeps its walk state in `zyl_ref` cells and searches liveness only where a decision needs it, so a round allocates nothing per node. A new fixpoint should do the same.
- A new runtime helper needs three things: the C function, its entry in the runtime's `X(...)` table in `actor_runtime.c` (which is what `zyl_runtime_export_p` consults, and what `zyl eval` and the REPL call through), and a signature in `stdlib/compiler/ffi_sigs.zyl` (without one, every `ffi-call` of it is `E_CANNOT_INFER`; the interpreter also takes a String or Float result type from it). If the compiler itself calls the helper, land it in two steps ([boot-fixed-point-workflow](boot-fixed-point-workflow.md)).

## See Also

- [pass-avoid-repeated-subtree-work](pass-avoid-repeated-subtree-work.md)
- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
