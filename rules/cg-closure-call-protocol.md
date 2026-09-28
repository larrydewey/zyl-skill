# cg-closure-call-protocol

> Calls through a local holding a function value go through the stack machine's `cg-call-indirect`, which passes one extra trailing argument: the closure env, or 0 for a plain code address. A code address is anything whose first word is not the closure tag, so no function may begin with the tag's eight bytes.

## Why It Matters

A plain function ignores the extra argument (the caller owns every SysV argument slot), so any function value can be passed anywhere and closure arity is unlimited. The tag test distinguishes a closure block (first word = `ic-closure-magic`, 2051230803) from a code address (anything else: a function's first eight bytes are its own instructions, `push rbp` or, for a frameless native function, its register pushes or its body, and never the tag's `53 4C 43 7A 00 00 00 00`). A value below `0x1000` exits with `zyl: invalid callee address`. A call by name to something that is neither a local nor a known function is a located `E_UNBOUND_VARIABLE` from the type pass (codegen's `cg-call-user` keeps a located backstop).

The native backend has no indirect call: `ml-ok` rejects an `ICall` whose name is a local, so a function that calls through a function value is compiled by the stack machine. It can still **take** a function's address (`MFnRef`, `lea rax, [rip+sym]`), and a native function is a valid target of an indirect call (a lifted closure's trailing `_clos_env` parameter is an ordinary parameter; a plain function ignores the extra argument).

```asm
    mov rax, [rbp-56]
    mov r11, 2051230803
    cmp qword ptr [rax], r11
    jne .L225
    mov rax, [rax+16]     ; env
    jmp .L226
.L225:
    xor eax, eax          ; plain function: 0
.L226:
    sub rsp, 8
    mov [rsp], rax        ; the env is the last scratch argument
    ...
    mov r10, [rbp-56]
    mov r11, 2051230803
    cmp qword ptr [r10], r11
    jne .L227
    mov r10, [r10+8]      ; code
.L227:
    call r10              ; or, in tail position: restore, leave, jmp r10
```

## Notes

- Tail calls through a function value reuse the frame too (`cg-tail-indirect`), when the callee's stack arguments (n + 1 with the env) fit this function's incoming stack-argument area.
- Frameless native functions (2026-09-28: no `push rbp` when there are no spills, blocks, regions or C calls) are fine: the test is equality with the tag word, and emitted code never starts with it. Emitting data at a function's label would break it.

## See Also

- [icnf-lowering-map](icnf-lowering-map.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
