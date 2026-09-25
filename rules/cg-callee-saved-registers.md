# cg-callee-saved-registers

> Every generated function preserves each callee-saved register it touches: the stack machine uses and saves only `rbx` and `r12`; the native backend allocates `rbx` and `r12`–`r15` and saves exactly the ones it used. Hand-written sequences use only the scratch registers.

## Why It Matters

Zyl functions are also called **from C**: `main` under `zyl_call_on_big_stack`, test functions under `zyl_run_tests`, actor entries, `qsort` comparators, callbacks on the FFI worker. An optimized C caller keeps live values in callee-saved registers, and a native-backend caller keeps its own values live across calls in `rbx`/`r12`–`r15`. A function that clobbers one without restoring it corrupts its caller, and the crash shows up far from the cause.

The two backends (both in `codegen.zyl`) keep the same contract differently:

| Backend | Callee-saved registers used | Saved where |
|---|---|---|
| stack machine (`cg-function-stack`) | `rbx` (variant block pointer), `r12` (saved `rsp` around each aligned C call) | always both, in the two lowest frame words (`cg-save-callee-saved` / `cg-restore-callee-saved`) |
| native (`mb-function`) | whatever linear scan gave a value live across a call, from `rbx r12 r13 r14 r15` | only those (`mb-used-callee`), in that order: pushed right after `rbp` when the function has no region words, else stored just below the 48 region bytes; restored by loads from the same slots before `ret` or a tail `jmp` |

```asm
; native backend, fib: two values cross a call
zy_local_x2Fmain_0__t3__fib:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    ...
    mov rbx, qword ptr [rbp-8]
    mov r12, qword ptr [rbp-16]
    mov rsp, rbp
    pop rbp
    ret
```

## Registers in the native backend

| Role | Registers |
|---|---|
| allocatable, caller-saved (values not live across a call) | `rsi rdi r8 r9 r10` |
| allocatable, callee-saved (values live across a call) | `rbx r12 r13 r14 r15` |
| scratch, never allocated | `rax rcx rdx r11` |

Any allocatable register may hold a live value at any instruction, so an emission sequence (`mb-emit-one` and its helpers) may freely use only `rax rcx rdx r11`. A sequence that has to call C in the middle of a body without being an `MCall` saves the caller-saved allocatable registers itself, as `mb-alloc-rax` does:

```asm
    push rsi
    push rdi
    push r8
    push r9
    push r10
    sub rsp, 8            ; 48 bytes: the frame stays 16-byte aligned
    mov rdi, 24
    call zyl_heap_alloc
    add rsp, 8
    pop r10
    ...
```

## Bad

```lisp
; an MIR emission helper borrowing rsi: it may hold a live vreg
(cg-emit-line st "    mov rsi, [rdx+8]")
; a stack-machine sequence using r13 without extending the save/restore pair
(cg-emit-line st "    mov r13, rax")
```

## Good

```lisp
(cg-emit-line st "    mov rdx, [rdx+8]")       ; scratch only
```

## Notes

- The region release on exit (`mb-region-release`) uses `r10` and `r11`; it runs only after the result is in `rax`, or after a tail call's arguments are staged in frame slots, when no vreg is live.
- A new instruction that calls C must also be in `mi-is-call` (so values live across it get callee-saved registers) and `mb-any-c-call` (so the frame is aligned): see [cg-native-backend-mir](cg-native-backend-mir.md).

## See Also

- [cg-native-backend-mir](cg-native-backend-mir.md)
- [cg-c-call-alignment](cg-c-call-alignment.md)
- [cg-stack-machine-fallback](cg-stack-machine-fallback.md)
