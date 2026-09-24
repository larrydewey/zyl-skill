# cg-c-call-alignment

> Route every C call through `cg-ext-call-aligned`, which forces 16-byte `rsp` alignment at the `call`.

## Why It Matters

SysV requires `rsp % 16 == 0` at `call`. The backend cannot guarantee it by construction (bodies run at 8 mod 16 and pushes shift it). Most C functions don't care; those using `movaps` fault — the REPL's `tcsetattr` crashed on its second call only, because the two call sites sat at different depths. Arity ≤ 6:

```asm
    mov r12, rsp
    and rsp, -16
    call zyl_cstr_len
    mov rsp, r12
```

Arity 7+: stack args must sit at `[rsp]`, so they are copied into a fresh aligned block:

```asm
    mov r12, rsp
    sub rsp, 24           ; 8 * stack-arg count
    and rsp, -16
    mov r10, [r12+0]
    mov [rsp+0], r10
    ...
    call snprintf
    mov rsp, r12
```

`print` (`printf`) and variant allocation (`zyl_heap_alloc`) use the same sequence.

## See Also

- [cg-callee-saved-rbx-r12](cg-callee-saved-rbx-r12.md)
- [boot-even-field-parity](boot-even-field-parity.md)
