# cg-symbols-and-entry

> User functions are labelled by mangled canonical keys (`zy_...`), the user entry is `_ZYL_main`, and every program shares one C `main` stub that runs it on a huge stack.

## Why It Matters

Reading assembly and link errors requires knowing the naming. `ffi-call` targets are sanitized to `[A-Za-z0-9_]`. A handful of recognized-by-spelling names keep the older `_ZYL_` + sanitized form.

```asm
main:
    push rbp
    mov rbp, rsp
    call zyl_save_args        ; argc/argv for zyl_argc / zyl_arg_str
    call zyl_ensure_arenas    ; heap + pin arenas
    lea rdi, [rip+_ZYL_main]
    call zyl_call_on_big_stack
    pop rbp
    ret
```

`zyl_call_on_big_stack` runs `_ZYL_main` on a pthread whose stack is an `mmap` reservation of 64 GiB (falling back to 16, 4, 1 GiB) with a guard page, and returns its value as the exit code. Literals go to `.rodata` (`.string`, `.double`). Output begins with `.file "<basename>.zyl"` so links are byte-identical.

## See Also

- [pkg-canonical-keys](pkg-canonical-keys.md)
- [fn-main-and-exit-status](fn-main-and-exit-status.md)
