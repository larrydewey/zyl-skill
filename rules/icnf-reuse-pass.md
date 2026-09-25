# icnf-reuse-pass

> In-place reuse (`reuse.zyl`, after region inference) lets a construction take the block of a value that is provably unique and dead; it only records decisions (`icnf-reuse`, owning clones `f~own`), the native backend acts on them, and every extension must fail toward allocating.

## Why It Matters

Values are immutable, so an update such as `(LCons v t)` built from `l` allocates a new record. When the old value is unique (nothing else refers to it) and dead (nothing uses it afterwards), nothing can observe the old block, so the new record can be written into it: Koka's Perceus "functional but in place", decided statically. Getting uniqueness or death wrong is silent memory corruption (a live value overwritten), so every condition is conservative.

`ru-reuse` runs last in `lower-after-mono` (`pipeline.zyl`), after `rg-regions`, and:

1. Collects which functions are opaque (named as a value, called through a closure, or `main`): they get no owning clone.
2. Computes per-function facts to a fixpoint over the whole program, in program order (at most 20 rounds; a round re-walks only functions that call one whose facts changed): which parameters an owning clone reuses (a bit mask, which only grows) and whether results are fresh.
3. Rewrites: records on each chosen `IVariant` the variable whose block it takes (`icnf-reuse-set`), redirects calls whose owned, dead arguments sit in reused positions to `f~own`, and appends each clone `f~own` (with a copy of `f`'s region summary) after `f`.

## Conditions (all checked in reuse.zyl)

- **Ownership.** Bound to a fresh value (a construction, or a call of a function whose results are fresh) or a parameter of an owning clone. Any use that could keep a second reference taints it: storing it in a variant, binding it to another name, `set!`, passing it where the callee's region summary lets the parameter escape, to an FFI entry not known to keep nothing, to a closure, or using it inside a lambda, `try` or region scope. Reading fields (`match`), comparing it, and passing it to a parameter with summary 0 do not.
- **Death.** Not used after the construction in evaluation order, and the construction is not inside a loop the variable was bound outside of.
- **Region.** At least one field of the new record is a pointer read out of the old one (a match binder of it). Region inference's classes are field-insensitive, so the two share a class and a region: the old block lives exactly as long as the new value may.
- **Heap block.** The variable's type is a program ADT (`icnf-adts`); stack variants (no size header) are never owned.

At run time the block is taken only when its size header covers the new record; otherwise the construction allocates as usual (`MReuse`, [cg-native-backend-mir](cg-native-backend-mir.md)).

## Bad

```lisp
; reusing where a record field is not read out of the old value:
; bump's new P shares no pointer with p, so no class ties p's block to the result's region
(defn bump (p) (match p (P x y (P (+ x 1) y))))     ; P of two Ints: reuse-mask=0 (correctly)
```

## Good

```lisp
; the new LCons keeps t, a binder of l: set-head~own takes l's block
(defn set-head (l v) (match l (LNil LNil) (LCons _ t (LCons v t))))
(defn spin (l n) (if (= n 0) l (spin (set-head l n) (- n 1))))   ; calls set-head~own
```

## Notes

- `ZYL_REUSE=0` turns the pass off. `ZYL_REUSE_DEBUG=1` prints the rounds and, per function, `reuse-mask`, `fresh`, `clone-fresh` and `opaque` on stderr.
- Only the native backend honours `icnf-reuse`; the stack machine and the interpreter ignore it and allocate. A clone is still emitted for either.
- Clones are made only for functions of at most 150 nodes (`ru-clone-limit`) with an ADT parameter.
- The walk keeps its state in cells so a fixpoint round allocates nothing per node ([pass-no-allocation-in-lookups](pass-no-allocation-in-lookups.md)).
- A new `Icnf` node needs cases in `ru-walk-node`, `ru-occurs`, `ru-calls-any` and `ru-opaque-refs` ([icnf-new-form-needs-case](icnf-new-form-needs-case.md)).
- The ICNF hash of a package build does not cover `icnf-reuse` (not printed), only the clone functions.

## See Also

- [icnf-regions-are-a-rewrite](icnf-regions-are-a-rewrite.md)
- [icnf-optimizer-scope](icnf-optimizer-scope.md)
- [pass-conservative-failure-direction](pass-conservative-failure-direction.md)
