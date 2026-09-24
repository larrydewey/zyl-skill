# cg-emission-appends

> Emit assembly by appending to the text buffer (`zyl_str_append` via `cg-emit*`), never by copying.

## Why It Matters

Symptom "output truncated to the last emitted line" means something used a copy (`strcpy`) instead of an append for buffer emission. `buf-append` semantics (append at `strlen(dst)`) are what the emitter depends on.

## See Also

- [proj-buf-append-appends](proj-buf-append-appends.md)
