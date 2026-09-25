# pass-avoid-repeated-subtree-work

> Visit each subtree once per pass; a pass that re-walks a subtree per visit goes exponential in nesting depth.

## Why It Matters

Type inference once inferred the last statement of every body twice. With right-nested bodies that doubled cost per statement, and a boot stage took about ten minutes; removing the duplicate made the whole fixed-point check take seconds. When compile time grows with nesting depth, suspect this first.

## Notes

- Whole-program fixpoints multiply the cost: region summaries and reuse facts are recomputed per round, so a per-node cost that is fine once can dominate. The reuse pass re-walks only functions that call one whose facts changed last round, and searches liveness only at a candidate site; follow that shape.
- Deep recursion over trees is normal (non-tail recursion; the big stack absorbs it).
- Assume what you allocate is kept: compiler data mostly escapes into results or global tables, so it lands in the heap, which is never freed during the compile (`E_OUT_OF_MEMORY` past the budget). Per-call regions reclaim only temporaries proven to die in their call.

## See Also

- [boot-moderate-bodies](boot-moderate-bodies.md)
- [pass-no-allocation-in-lookups](pass-no-allocation-in-lookups.md)
