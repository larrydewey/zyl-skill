# cg-stack-machine-fallback

> Know the stack machine as the fallback backend: every function the native backend declines (`mb-eligible` false, about 5% of the compiler's own functions, or all of them under `ZYL_MIR=0`) is compiled to code over `rax` with `rbp`-relative slots and no register allocator.

## Why It Matters

`cg-function` sends a function to `mb-function` (the native backend, [cg-native-backend-mir](cg-native-backend-mir.md)) when `mb-eligible` accepts it, and to `cg-function-stack` otherwise. The fallback takes anything the native backend does not: `print`, Float or String or ADT operators, calls through function values and closures, `try`, `with-region` scopes, frame wiping for `Secret`, more than six parameters or arguments. Both backends share one ABI, so they call each other freely; a bug that only reproduces with `ZYL_MIR=0` (or only without it) points at one of them.

Every expression leaves its value in `rax`. A binop evaluates the left operand, pushes it, evaluates the right, moves it to `rcx`, pops the left into `rax`, combines. Parameters are spilled to `[rbp-8]`, `[rbp-16]`, ... (to `[rbp-56]`, ... in a function with region words, see Frame); stack args 7+ are read from `[rbp+16]`.... `rax rcx rdx r10 r11 r12 rbx` are fixed scratch in particular sequences. Output is GNU assembler `.intel_syntax noprefix`, built in a fixed 64 MiB buffer (`E_CODEGEN_BUFFER_FULL` past 63 MiB; writes bounds-checked via `zyl_str_append_capped`).

```asm
    mov rax, [rbp-8]      ; left
    push rax
    mov rax, [rbp-16]     ; right
    mov rcx, rax
    pop rax
    add rax, rcx
```

## Frame

```
[rbp+16].. stack args 7+   [rbp+8] return   [rbp] saved rbp
[rbp-8]..  params, then one slot per let / match binding / loop value / stack-variant word
lowest two words: saved rbx, r12      frame size 24 + 16*(1 + nslots/2), i.e. 16n+8
```

A function region inference flags (`icnf-region` on the `IFn` = flags + 4: bit 0 has a frame region, bit 1 keeps the result region) has six words above its parameters, the layout the runtime and the native backend share:

```
[rbp-8]  saved rax          [rbp-16] result region (zyl_cur_region at entry)
[rbp-48] region header: prev, bump, end, blocks  (four words)
[rbp-56].. params, then locals
```

Entry pushes the header on the thread-local `zyl_region_top` chain inline; exit and every tail jump pop it inline and call `zyl_region_free` only if a block was taken. Before each call the site's region (frame header, saved result region, or 0 for the heap) is stored in `zyl_cur_region` (`fs`-relative TLS). An `IVariant` at a region site allocates through `zyl_ralloc(size, region)`; a heap site calls `zyl_heap_alloc`. A `with-region` scope (`IRegion`) uses the same four-word header layout, marked by the low bit of `blocks`.

## Notes

- `while` keeps its value in a slot initialized to 0; empty `begin` is `xor eax, eax`.
- Match: push scrutinee, compare tag word per arm; wildcard (tag -1) skips the compare; no match → 0.
- Emission uses a `CGState` text buffer via `cg-emit`/`cg-emit-line`/`cg-emit-int`; labels `cg-label-new`; rodata `cg-with-rodata`. The native backend emits into the same buffer with the same label counter.
- The stack machine ignores the reuse pass's decisions (`icnf-reuse`): it always allocates.

## See Also

- [cg-native-backend-mir](cg-native-backend-mir.md)
- [cg-callee-saved-registers](cg-callee-saved-registers.md)
- [cg-call-arg-staging](cg-call-arg-staging.md)
