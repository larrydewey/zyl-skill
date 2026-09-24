# type-value-representation

> Reason about values as single 64-bit words with a fixed, documented layout.

## Why It Matters

The uniform one-word representation explains most of Zyl's behavior: why unannotated functions are polymorphic, why generics need no per-type copies, why printing and comparing depend on known kinds, and what C sees across the FFI.

| Type | Word holds |
|---|---|
| `Int` | two's-complement integer, untagged |
| `Float` | IEEE-754 binary64 bit pattern (unboxed; moved to xmm only for arithmetic) |
| `Bool` | 0 or 1 (`print true` shows `1`) |
| `Byte` | 0–255 (unifies with Int) |
| `String` | pointer to NUL-terminated UTF-8 (literals in `.rodata`) |
| struct / ADT | pointer to `[hidden size word][tag][field0][field1]...`, 8 bytes per slot |
| non-capturing closure | code address |
| capturing closure | pointer to `[magic tag][code][env]`; env block holds captured words |
| `ByteBuf` / `ByteSlice` | pointer to a runtime header (magic, data ptr, len, cap) |
| `Vec` / `Map` / `Set` | ordinary structs (`ptr len cap arena`) |
| `Unit` | meaningless word (often 0) |

## Notes

- ADT tags: 0-based in declaration order per `deftype`; struct tags from a global counter starting at 100000.
- Calling convention: SysV registers `rdi rsi rdx rcx r8 r9`, then stack; result in `rax` (floats too, as bits).
- Layouts are implementation details: do not write C that depends on them.

## See Also

- [own-let-copies-word](own-let-copies-word.md)
- [ffi-int64-only-no-floats](ffi-int64-only-no-floats.md)
