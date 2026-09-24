# bits-regions-unenforced-use-pin

> Write `Pin` for any buffer whose address you take or share, even though region rules are not enforced yet.

## Why It Matters

The region argument is recorded in the type (`ByteBuf Region`) but the runtime gives every region the same stable, zero-initialized heap allocation, and none of the designed checks run:

| Rule | Designated error | Today |
|---|---|---|
| `bytebuf-ptr` only on Pin buffers | `E_BYTEBUF_NOT_PIN` | not checked |
| Stack buffer may not escape | `E_STACK_BYTEBUF_RETURN` | not checked |
| Global buffer immutable | `E_GLOBAL_BYTEBUF_MUT` | not checked |
| CAS only on Pin memory | `E_ATOMIC_ABA` | not checked |

Writing the correct region now keeps the program valid when checks land.

## Good

```lisp
(let shared (bytebuf Pin 64)
  (ffi-call "c_fill" (bytebuf-ptr shared) 64 1000))
```

## See Also

- [own-regions-status](own-regions-status.md)
