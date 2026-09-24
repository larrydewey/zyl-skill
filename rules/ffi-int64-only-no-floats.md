# ffi-int64-only-no-floats

> Declare C functions called from Zyl with `int64_t` and pointer parameters and results only; never pass a `Float` to a C `double`.

## Why It Matters

Every argument travels as one 64-bit word in the SysV **integer** registers (`rdi rsi rdx rcx r8 r9`, then stack), and the result is read from `rax`. The compiler never loads `xmm` registers for arguments and never reads `xmm0`. A C `double f(double)` receives garbage and its result is lost (e.g. `2.0` comes back as `4611686018427387904`). `al` is not set for variadic calls: `printf` with integer args happens to work, with floats it does not.

## Bad

```c
double scale(double x);          /* receives garbage from Zyl */
```

## Good

```c
#include <stdint.h>
#include <string.h>
int64_t zyl_factorial(int64_t n) { return n <= 1 ? 1 : n * zyl_factorial(n - 1); }

/* float wrapper: bits in, bits out */
int64_t scale_bits(int64_t xbits) {
    double x; memcpy(&x, &xbits, 8);
    double r = x * 2.0;
    int64_t out; memcpy(&out, &r, 8);
    return out;
}
```

## What C receives

| Zyl value | C sees | Declare as |
|---|---|---|
| Int, Bool | the integer (0/1) | `int64_t` |
| String | pointer to NUL-terminated bytes | `const char *` |
| allocator buffer | raw pointer (an Int in Zyl) | `char *`/`void *` |
| struct/ADT | pointer to `[tag][field0]...` (internal layout) | `const int64_t *` — avoid depending on it |
| `(ffi-pin v)` | pointer to an 8-byte slot holding v | `const int64_t *` |
| Float | IEEE bits in an integer register | only via bit wrappers |

## Notes

- There is no `zyl_ffi.h` header; use `<stdint.h>`.
- More than six arguments work in compiled code (arity 7+ copies stack args into an aligned block). The interpreter supports at most six.
- The pinnability check exists but the current type checker does not reach every argument: `(ffi-call "abs" (Some 1) 1000)` compiles and passes an address.

## See Also

- [ffi-pin-passes-pointer](ffi-pin-passes-pointer.md)
- [type-value-representation](type-value-representation.md)
