# pass-avoid-repeated-subtree-work

> Visit each subtree once per pass; a pass that re-walks a subtree per visit goes exponential in nesting depth.

## Why It Matters

Type inference once inferred the last statement of every body twice. With right-nested bodies that doubled cost per statement, and a boot stage took about ten minutes; removing the duplicate made the whole fixed-point check take seconds. When compile time grows with nesting depth, suspect this first.

## Notes

- Deep recursion over trees is normal (no TCO; the big stack absorbs it).
- Allocate from the per-compile arena; it is never freed during the compile (`E_OUT_OF_MEMORY` past the budget).

## See Also

- [boot-moderate-bodies](boot-moderate-bodies.md)
