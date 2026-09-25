# type-value-representation

> Reason about values as single 64-bit words with a fixed layout; the type checker, not the word, says what a value is.

## Why It Matters

The uniform one-word representation explains most of Zyl's run-time behavior: why unannotated functions share one body, why printing and comparing need the static type (the word alone does not say whether it is an Int or a String pointer), and what C sees across the FFI. Sound type checking guarantees that no word is ever read at the wrong representation ([type-sound-checking](type-sound-checking.md)).

| Type | Word holds |
|---|---|
| `Int` | two's-complement integer, untagged |
| `Float` | IEEE-754 binary64 bit pattern (unboxed; moved to `xmm` only for arithmetic) |
| `Bool` | 0 or 1. `(print true)` prints `1`; `Show.show` and container printing give `true`/`false` (`(print (list true))` is `[true]`) |
| `Unit` | a word with no meaning; the literal `unit` is 0. `print` of a Unit expression prints whatever word it happens to be (`(print unit)` is `0`, a `set!` may show the stored value); `Show` has no Unit impl |
| `String` | pointer to NUL-terminated UTF-8 (literals in `.rodata`) |
| struct / ADT | pointer to `[hidden size word][tag][field0][field1]...`, 8 bytes per slot, in a frame region, a result region or the heap ([own-regions-status](own-regions-status.md)) |
| non-capturing closure | code address |
| capturing closure | pointer to `[magic tag][code][env]`; the env block holds captured words |
| `ByteBuf` / `ByteSlice` | pointer to a runtime header (magic, data ptr, len, cap) |
| runtime handles (`Arena`, `Array`, `Ptr`, ...) | an opaque runtime pointer or index |
| `Vec` | ADT `(VecC (Array T) Int Arena)`: storage array, length, arena |
| `StrView`, `Slice` | ADTs holding their base and an offset and length ([data-views-and-slices](data-views-and-slices.md)) |

## Notes

- There is no `Byte` type: byte loads return `Int` (0–255), and `(b Byte)` is an unconstrained type variable.
- ADT tags are 0-based in declaration order per `deftype`; struct tags come from a global counter starting at 100000.
- The hidden size word lets structural equality and in-place reuse ([data-collections-persistent](data-collections-persistent.md)) know the block size.
- Calling convention: SysV registers `rdi rsi rdx rcx r8 r9`, then stack; result in `rax` (Floats too, as bits). Both backends (MIR and the stack machine) use it.
- Layouts are implementation details: do not write C that depends on them.

## See Also

- [own-let-copies-word](own-let-copies-word.md)
- [ffi-extern-word-sized-types](ffi-extern-word-sized-types.md)
