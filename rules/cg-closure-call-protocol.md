# cg-closure-call-protocol

> Calls through a local holding a function value go through `cg-call-indirect`, which passes one extra trailing argument: the closure env, or 0 for a plain code address.

## Why It Matters

A plain function ignores the extra argument (the caller owns every SysV argument slot), so any function value can be passed anywhere and closure arity is unlimited. The tag test distinguishes a closure block (first word = `ic-closure-magic`, 2051230803) from a code address (first byte is this compiler's `push rbp`, 0x55). A value below `0x1000` exits with `zyl: invalid callee address`. A call by name to something that is neither a local nor a known function is a located `E_UNBOUND_VARIABLE` from `cg-call-user`.

```asm
    mov rax, [rbp-24]
    mov r11, 2051230803
    cmp qword ptr [rax], r11
    jne .L7
    mov rax, [rax+16]     ; env
    jmp .L8
.L7:
    xor eax, eax          ; plain function: 0
.L8:
    ...
    mov r10, [rbp-24]
    cmp qword ptr [r10], r11
    jne .L9
    mov r10, [r10+8]      ; code
.L9:
    call r10
```

## See Also

- [icnf-lowering-map](icnf-lowering-map.md)
