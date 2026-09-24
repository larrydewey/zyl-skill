# cg-call-arg-staging

> Preserve `cg-call-args`' staging order: parity pad, each argument evaluated left to right into its own scratch slot, stack args copied, registers loaded, `call`, one cleanup `add`.

## Why It Matters

Staging guarantees strict left-to-right evaluation and that evaluating one argument cannot clobber another already computed. Pad and pops must agree; a mismatch shows up as garbage in a variable or misaligned C calls.

```asm
    sub rsp, 8            ; parity pad (odd pushed-word count)
    mov rax, 1
    sub rsp, 8
    mov [rsp], rax        ; arg 1
    mov rax, 2
    sub rsp, 8
    mov [rsp], rax        ; arg 2
    mov rdx, [rsp+0]      ; ... load registers from scratch
    mov rsi, [rsp+8]
    call zy_local_x2Fmain_0__add__add3
    add rsp, 32
```

## Notes

- Division: `cqo` before `idiv` (stale `rdx` gives SIGFPE); remainder via `mov rax, rdx`. Division by zero is unchecked.
- Comparisons: `cmp`, `setX al`, `movzx rax, al`. Shifts use the branchless defined-count sequences.
- Floats: moved to `xmm0/xmm1` only for `addsd/subsd/mulsd/divsd/comisd`.

## See Also

- [cg-stack-machine](cg-stack-machine.md)
- [det-left-to-right](det-left-to-right.md)
