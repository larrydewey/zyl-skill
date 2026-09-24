# cg-callee-saved-rbx-r12

> Use only `rbx` and `r12` among callee-saved registers in new codegen sequences, or extend `cg-save-callee-saved`/`cg-restore-callee-saved` first.

## Why It Matters

Zyl functions are also called **from C**: `main` under `zyl_call_on_big_stack`, test functions under `zyl_run_tests`, actor entries, `qsort` comparators. An optimized C caller keeps live values in callee-saved registers. Every prologue saves `rbx` (variant block pointer) and `r12` (saved `rsp` around C calls) in the two lowest frame words and the epilogue restores them. A new sequence using `r13`–`r15` without extending save/restore corrupts C callers — crashes far from the cause.

```asm
name:
    push rbp
    mov rbp, rsp
    sub rsp, 120
    mov [rbp-120], rbx
    mov [rbp-112], r12
    ...
    mov rbx, [rbp-120]
    mov r12, [rbp-112]
    mov rsp, rbp
    pop rbp
    ret
```

## See Also

- [cg-c-call-alignment](cg-c-call-alignment.md)
