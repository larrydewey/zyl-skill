# cg-c-call-alignment

> Keep `rsp` 16-byte aligned at every call into C: the stack machine realigns around each C call (`cg-ext-call-aligned`); the native backend aligns its frame once, in the prologue, and only when the function calls C.

## Why It Matters

SysV requires `rsp % 16 == 0` at `call`. Most C functions don't care; those using `movaps` fault — the REPL's `tcsetattr` crashed on its second call only, because the two call sites sat at different depths, and `snprintf` with SSE arguments faulted the same way.

## Stack machine

It cannot guarantee alignment by construction (bodies run at 8 mod 16 and every argument or field push shifts it), so every C call goes through `cg-ext-call-aligned`. Arity ≤ 6:

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

`print` (`cg-print-call`, `printf`), variant allocation (`cg-variant`: `zyl_ralloc` at a region site, `zyl_heap_alloc` at a heap site) and `setjmp` for `try` use the same save/`and`/restore idiom.

## Native backend

A native function's frame (saved registers, spill slots, staging words, stack-variant blocks) is rounded to a multiple of 16, so once `rsp` is aligned in the prologue it stays aligned for the whole body, and C calls need no per-call sequence. `mb-needs-align` emits `and rsp, -16` in the prologue only when the function has region words or contains an instruction that calls C (`mb-any-c-call`: a runtime `MCall`, `MAlloc`, `MArr` whose slow path calls the runtime, `MRegionCycle`). A function that calls only Zyl functions skips it: each callee aligns its own frame (native) or each of its own C calls (stack machine).

```asm
zy_local_x2Fmain_0__t3__slen:
    push rbp
    mov rbp, rsp
    push rbx
    and rsp, -16
    sub rsp, 16
    ...
    call zyl_cstr_len          ; no save/restore around it
```

Mid-body sequences that push must push an even number of words; the allocation slow path pushes five registers plus `sub rsp, 8`.

## Notes

- A new native instruction that calls C must be added to `mb-any-c-call`, or the function may call C misaligned when it is entered from a stack-machine caller (whose calls to Zyl functions only preserve whatever parity happened to hold). As of 2026-09-25 `MReuse`, whose slow path calls `zyl_ralloc`/`zyl_heap_alloc`, is not in that list: a function whose only allocation is a reuse site is aligned only because it also has region words (with `ZYL_REGIONS=0` such a function gets no `and rsp, -16`).
- Native calls take at most six arguments (`ml-ok`), so the arity-7 copy never arises there.

## See Also

- [cg-callee-saved-registers](cg-callee-saved-registers.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
- [boot-field-parity-lifted](boot-field-parity-lifted.md)
