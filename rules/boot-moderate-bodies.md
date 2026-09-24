# boot-moderate-bodies

> Keep compiler function bodies moderate and flat; prefer short `let` chains and helpers over deep nesting.

## Why It Matters

`cg-function` sizes each frame from its actual slots (parameters + one per `let`/pattern binding + 8 headroom, 8 bytes each, rounded to 16n+8), so frame size is no longer the old uniform worst case. Deep recursion relies on the big-stack worker (`zyl_call_on_big_stack`; generated entry stubs route `main` through it). Deeply nested match/let code is where residual codegen bugs have historically lived, and where compile-time cost blows up (see [pass-avoid-repeated-subtree-work](pass-avoid-repeated-subtree-work.md)).

## See Also

- [boot-lifted-constraints](boot-lifted-constraints.md)
