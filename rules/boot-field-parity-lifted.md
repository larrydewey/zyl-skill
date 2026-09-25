# boot-field-parity-lifted

> Do not pad record types to an even field count: odd counts in nested constructions compile correctly in both backends. The even `CheckState` in `sexp_balance.zyl` is a leftover workaround, not a rule.

## Why It Matters

`cg-variant`'s alignment padding for a nested variant-construction argument was once computed from the current call's own field count, not the caller's already-pushed argument count, so an odd count could misalign the stack. That was the confirmed cause of a segfault in `sexp_balance.zyl`'s `CheckState`, which was cut to an even field count, and its construction site pre-binds `Balanced` with a `let` (both explained in comments there).

The mechanism is gone. The stack machine's `cg-variant` now saves `rsp` in `r12`, aligns with `and rsp, -16` around `zyl_ralloc`/`zyl_heap_alloc` and restores it, with no parity computation; every C call is realigned the same way (`cg-ext-call-aligned`); and the native backend allocates with aligned frames ([cg-c-call-alignment](cg-c-call-alignment.md)). Verified 2026-09-25, native and `ZYL_MIR=0`: a nine-field record built with nested constructions as arguments (`(CS 1 (str-length "abc") (Cons 1 Nil) 1 0 0 0 0 (Bad (str-length "hello")))`) and read back gives the right fields.

## Good

- Give a record the fields it needs. Nested constructions as constructor arguments are fine.
- Removing the leftover workaround in `sexp_balance.zyl` changes the compiler's own output: reseed and run the full suite ([boot-fixed-point-workflow](boot-fixed-point-workflow.md)).

## See Also

- [cg-c-call-alignment](cg-c-call-alignment.md)
- [boot-lifted-constraints](boot-lifted-constraints.md)
