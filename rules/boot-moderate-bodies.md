# boot-moderate-bodies

> Keep compiler function bodies moderate and flat; prefer short `let` chains and helpers over deep nesting.

## Why It Matters

Frames are sized from what a function actually uses: the stack machine from its slots (parameters + one per `let`/pattern binding + 8 headroom, 8 bytes each, rounded to 16n+8), the native backend from the callee-saved registers it saves plus its spill slots, staging words and stack-variant blocks. Deep recursion relies on the big-stack worker (`zyl_call_on_big_stack`; generated entry stubs route `main` through it).

Size still matters, for other reasons:

- Compile time. Several passes do work per node per round (the reuse and region fixpoints, the inliner's size counts); a pass that re-walks subtrees goes exponential in nesting ([pass-avoid-repeated-subtree-work](pass-avoid-repeated-subtree-work.md)).
- Register pressure. Linear scan has ten registers; a long body with many values live across calls spills to frame slots.
- Optimization reach. Only small functions are inlined (`ZYL_INLINE_LIMIT`, default 6 nodes; leaves up to 18, into loops only) and cloned for reuse (at most 150 nodes).
- Backend choice. One unsupported node (`print`, a String or Float operator, a closure call, `try`) sends the whole function to the stack machine; keeping such code in its own small helper leaves the rest native.

Deeply nested match/let code is also where residual codegen bugs have historically lived.

## See Also

- [boot-lifted-constraints](boot-lifted-constraints.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
