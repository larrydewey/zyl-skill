# cg-call-arg-staging

> Evaluate call arguments strictly left to right and never let one argument's evaluation clobber another's value: the stack machine stages them in scratch slots (or takes its direct-register path when that is provably safe), the native backend evaluates them into fresh vregs and moves them into argument registers with one parallel move.

## Why It Matters

Staging guarantees strict left-to-right evaluation and that evaluating one argument (which may itself call) cannot overwrite an argument register already filled. Pad and pops must agree; a mismatch shows up as garbage in a variable or misaligned C calls.

## Stack machine (`cg-call-args`)

General path: parity pad, each argument evaluated left to right into its own scratch slot, stack args (7+) copied, registers loaded from scratch, `call`, one cleanup `add`:

```asm
    sub rsp, 8            ; parity pad (odd pushed-word count)
    ...                   ; arg 1 (a call)
    sub rsp, 8
    mov [rsp], rax
    ...                   ; arg 2 (a call)
    sub rsp, 8
    mov [rsp], rax
    mov rax, 4
    sub rsp, 8
    mov [rsp], rax        ; arg 3
    mov rdx, [rsp+0]      ; load registers from scratch
    mov rsi, [rsp+8]
    mov rdi, [rsp+16]
    call zy_local_x2Fmain_0__t8__add3_x7EInt_x2CInt_x2CInt
    add rsp, 32
```

Direct path (`cg-direct-args-ok`): at most six arguments, at most one of them more than a constant or local, and none containing a `set!`. The complex one is evaluated first, straight into its register, then the simple ones are moved in; the stack is lowered by the same amount so cleanup and alignment are unchanged.

## Native backend (`ml-args`, `mb-arg-moves`)

Each argument is lowered in order to a vreg; a local read is copied into a fresh vreg when a later argument contains a `set!` (`ml-operand` with `icnf-list-has-set`). At the `MCall` the vregs' locations are moved into `rdi rsi rdx rcx r8 r9` as one parallel move (`pm-sequence` in `mir.zyl`: a move whose destination no other pending move reads goes first; a cycle is broken through `rax`). Native calls have at most six arguments.

```asm
    mov rdi, rbx
    call zy_local_x2Fmain_0__t9__sq
    mov r13, rax              ; kept across the next call
    mov rdi, r12
    call zy_local_x2Fmain_0__t9__sq
    mov rsi, rax              ; parallel move into rdi rsi rdx
    mov rdi, r13
    mov rdx, rbx
```

## Notes

- Division: the stack machine and the native backend's general path use `cqo` before `idiv` (stale `rdx` gives SIGFPE); remainder via `mov rax, rdx`. Division by zero is unchecked. The native backend divides by a constant other than 0 and -1 without `idiv` (`mb-divmod-const`: a move for 1, a biased shift for a power of two, else a multiply by `zyl_div_magic`'s constant), with `idiv`'s results.
- Comparisons: `cmp`, `setX al`, `movzx rax, al`, or a conditional jump when the comparison is a condition. Shifts use the branchless defined-count sequences (`cg-bit-mnem`) in both backends.
- Floats (stack machine only): moved to `xmm0/xmm1` for `addsd/subsd/mulsd/divsd/comisd`.

## See Also

- [cg-stack-machine-fallback](cg-stack-machine-fallback.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
- [pass-evaluation-order-sets](pass-evaluation-order-sets.md)
- [det-left-to-right](det-left-to-right.md)
