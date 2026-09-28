# cg-symbols-and-entry

> User functions are labelled by mangled canonical keys (`zy_...`), the user entry is `_ZYL_main`, and every program shares one `main` stub that runs it on a huge stack; a freestanding binary starts at the runtime's `_start`.

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

A freestanding binary enters at `_start` (`start.s`), which calls the runtime's `zyl_rt_start` (argc/argv/envp/auxv, the static TLS block, fs) and then `main`; a hosted one enters through libc's crt. `zyl_call_on_big_stack` runs `_ZYL_main` on a thread whose stack is an `mmap` reservation of 64 GiB (falling back to 16, 4, 1 GiB) with a guard page, and returns its value as the exit code. Literals go to `.rodata` (`.string`, and a float as its exact bits, `.quad`). Output begins with `.file "<basename>.zyl"` so links are byte-identical.

## Reading labels

- `zy_local_x2Fmain_0__t9__add3` is `local/main@0::t9::add3`: `zyl_mangle_key` prefixes `zy_`, writes `@` as `_` and `::` as `__`, and escapes other punctuation as `_x<hex>` (`/` is `_x2F`, `-` is `_x2D`).
- A per-type instance made by the type pass is `f~T` (`sq~Int`, `add3~Int,Int,Int`), so its label ends `_x7EInt` (`~` is `_x7E`, `,` is `_x2C`). An unannotated function whose body uses a class such as `Num` is called through such instances.
- An owning clone made by the reuse pass is `f~own` (`_x7Eown`): [icnf-reuse-pass](icnf-reuse-pass.md).
- Both backends use the same function labels. Inside a native-backend function, local labels are `.L<k>_<n>` (`.L<k>_0` is the head a self tail call jumps to); the stack machine's are plain `.L<k>`.

## See Also

- [pkg-canonical-keys](pkg-canonical-keys.md)
- [fn-main-and-exit-status](fn-main-and-exit-status.md)
