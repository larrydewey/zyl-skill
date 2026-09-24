# cg-stack-machine

> Model codegen as a stack machine over `rax` with `rbp`-relative slots and no register allocator.

## Why It Matters

Every expression leaves its value in `rax`. A binop evaluates the left operand, pushes it, evaluates the right, moves it to `rcx`, pops the left into `rax`, combines. Parameters are spilled to `[rbp-8]`, `[rbp-16]`, ...; stack args 7+ copied from `[rbp+16]`.... `rax rcx rdx r10 r11 r12 rbx` are fixed scratch in particular sequences. Output is long but trivially deterministic. Output is GNU assembler `.intel_syntax noprefix`, built in a fixed 64 MiB buffer (`E_CODEGEN_BUFFER_FULL` past 63 MiB; writes bounds-checked via `zyl_str_append_capped`).

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
lowest two words: saved rbx, r12      frame size 16n+8
```

## Notes

- `while` keeps its value in a slot initialized to 0; empty `begin` is `xor eax, eax`.
- Match: push scrutinee, compare tag word per arm; wildcard (tag -1) skips the compare; no match → 0.
- Emission uses a `CGState` text buffer via `cg-emit`/`cg-emit-line`/`cg-emit-int`; labels `cg-label-new`; rodata `cg-with-rodata`.

## See Also

- [cg-callee-saved-rbx-r12](cg-callee-saved-rbx-r12.md)
- [cg-call-arg-staging](cg-call-arg-staging.md)
