# pass-conservative-failure-direction

> Make checks fail open (miss a diagnostic rather than reject valid code) and transformations fail safe (fall back to the always-correct path).

## Why It Matters

Every existing checker (mutability, capability, Secret, exhaustiveness) is syntactic and conservative in the permissive direction: a shape it does not understand is let through. Region inference proves a narrow property and otherwise keeps values on the heap. Type inference degrades to type variables. Follow the same direction so a new pass never breaks valid programs or the fixed point — and document what it misses, because users will assume full enforcement.

## See Also

- [icnf-regions-are-a-rewrite](icnf-regions-are-a-rewrite.md)
- [type-inference-does-not-reject](type-inference-does-not-reject.md)
