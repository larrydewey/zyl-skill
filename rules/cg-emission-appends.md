# cg-emission-appends

> Emit assembly by appending to the text buffer (`zyl_str_append_capped` via `cg-emit*`), never by copying.

## Why It Matters

Both backends write into the one `CGState` buffer (a 64 MiB `StrBuf`) through `cg-emit`, `cg-emit-line`, `cg-emit-lines` and `cg-emit-int`. The runtime keeps the buffer's length in its header, so an append is a `memcpy` at the end, and past the cap it panics (`codegen buffer limit exceeded`; the pipeline reports `E_CODEGEN_BUFFER_FULL` past 63 MiB). Symptom "output truncated to the last emitted line" means something used a copy (`strcpy`) instead of an append. Building an instruction string with `str-concat` first and emitting it once (as the `mb-*` emitters do) is fine; building the whole function text by concatenation is quadratic.

## See Also

- [proj-buf-append-appends](proj-buf-append-appends.md)
- [pass-no-allocation-in-lookups](pass-no-allocation-in-lookups.md)
