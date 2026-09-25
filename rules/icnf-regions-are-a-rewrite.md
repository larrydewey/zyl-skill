# icnf-regions-are-a-rewrite

> Region inference is two ICNF passes, the conservative `IStackVariant` rewrite and the whole-program `rg-*` classification into the `icnf-regions` side table; every extension must fail toward the heap.

## Why It Matters

`region_inference.zyl` runs in `pipeline.zyl` after inlining and optimization as `(rg-regions (ri-transform-fns opt-fns))`, and the reuse pass runs on its result:

1. **`ri-transform-fns`**: a let-bound `IVariant` becomes `IStackVariant` only when `ri-name-safe-in` (a total match over every `Icnf` constructor) proves every use is a `match` scrutinee or `print` argument and no nested `fn` references it.
2. **`rg-regions`**: classifies every allocation site (`IVariant`, region-aware runtime calls) and every call site as L (frame region), R (the caller's result region, passed in the thread-local `zyl_cur_region`) or H (heap). Levels belong to union-find object classes (runtime `zyl_uf_*`), field-insensitive; nodes the type pass proves `Int`/`Bool`/`Float` (`icnf-scalars`, from `ta-scalar`) never join a class. Per-function summaries (per parameter: 0 does not escape, 1 may reach the result, 2 escapes; bit 62: may allocate into its result region; kept by name in `rg-summaries`) are recomputed to a fixpoint and **joined** (per-parameter max) with the previous round, because the constraints are not monotone in the summary: plain replacement once cycled forever on `math-blake3`.

Results go in `icnf-regions` (`node_tables.zyl`; site: level + 1, i.e. 1 frame, 2 result, 3 heap, `4 + k` with-region scope `k`; function node: flags + 4, bit 0 has a frame region, bit 1 keeps the result region). `icnf_print` shows them as ` @r`, so the package-build ICNF hash covers region decisions. `rg-regions` also raises `E_REGION_ESCAPE` for Stack bytebufs and `with-region` values that escape.

Both backends read the same annotations: the stack machine through `cg-site-level`/`cg-region-flags`, the native backend as the region level on `MCall`/`MAlloc`/`MReuse` and the function's region flags (it declines sites of level 4 and up, which keep the stack machine).

Undershooting a level is silent memory corruption (a freed region still referenced); overshooting costs memory. So:

- Tail-call arguments are at least R (the frame is gone when the callee runs).
- Calls through a function value: arguments H, result joins the closure's class.
- Runtime functions are trusted only from the explicit `rg-ffi-kind` table (1 fresh result in `zyl_cur_region`, 2 result aliases the first argument, 0 keeps nothing; byte-buffer loads/stores/atomics have their own kinds). Anything unlisted keeps arguments and result in the heap. Foreign `ffi-call` arguments are H.
- A kind-1 runtime function must allocate only through `zyl_result_alloc`, and compiled code calls its `_r` entry point from annotated sites (both backends append `_r` when the level is above 0); adding a function to the table without that is a use-after-free.
- The reuse pass trusts the summaries too (a parameter with summary 0 is only read) and relies on field-insensitive classes to keep an old block and the record built into it in one region. A change that makes classes field-sensitive must revisit `reuse.zyl`.

## Bad

```lisp
; trusting a runtime function that keeps its argument (e.g. stores it in a table)
(if (str-eq s "zyl_smap_put") 0          ; claims "keeps nothing": the key's region is freed under the table
```

## Good

```lisp
; unlisted functions fall through to the heap default
(defn rg-ffi-kind (s)
  (if (str-eq s "zyl_cstr_concat") 1
  ...
  (rg-ffi-kind-bytes s)))                 ; unknown: heap
```

## Notes

- `ZYL_REGIONS=0` at compile time skips `rg-regions` entirely (every site H). Use it to bisect a suspected region bug.
- A new `Icnf` node needs a case in `ri-name-safe-in` and in `rg-expr-node` (see [icnf-new-form-needs-case](icnf-new-form-needs-case.md)).
- The interpreter ignores the annotations.

## See Also

- [own-stack-promotion](own-stack-promotion.md)
- [own-with-region](own-with-region.md)
- [icnf-reuse-pass](icnf-reuse-pass.md)
- [pass-conservative-failure-direction](pass-conservative-failure-direction.md)
