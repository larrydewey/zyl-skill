# boot-even-field-parity

> Give a record type that is passed as a constructor argument to another constructor call an even field count, until the workaround is re-tested.

## Why It Matters

`cg-variant`'s alignment padding for a nested variant-construction argument was computed from the current call's own field count, not the caller's already-pushed argument count; an odd count could misalign the stack. This was the confirmed root cause of a segfault in `sexp_balance.zyl`'s `CheckState` (see its header comment). Since then `cg-variant`'s `zyl_heap_alloc` call and every C call (`cg-ext-call-aligned`, 2026-09-23) realign `rsp` to 16 bytes, which addresses the mechanism — but the even-count workaround in `sexp_balance.zyl` has not been removed and re-tested.

## Good

Keep matching field-count parity on such types (add a padding field if needed) in compiler code; remove the workaround only with a boot + full regression run.

## See Also

- [cg-c-call-alignment](cg-c-call-alignment.md)
