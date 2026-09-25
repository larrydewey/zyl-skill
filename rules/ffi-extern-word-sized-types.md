# ffi-extern-word-sized-types

> Give every foreign function an `(extern ...)` signature built from word-sized types (Int, Bool, String, `Ptr`, `(Pin a)`, handles, `(Fn (A ...) R)`), declare the C side with `int64_t` and pointers, and cross a `Float` only as its bits.

## Why It Matters

A foreign symbol is typed by its `(extern "sym" (ParamType ...) ResultType)` declaration, and the timed FFI worker (`zyl_ffi_timed`) passes every argument as one 64-bit word in the SysV **integer** registers and reads the result from `rax`. No `xmm` register is ever loaded or read, so a C `double` parameter would receive garbage. The type checker therefore rejects `Float` anywhere in an extern (`E_TYPE_MISMATCH`, "uses the type Float, which cannot cross the C boundary"), and a type variable too (it would be a cast). Arguments are unified with the declared types, so `(ffi-call "abs" (Some 1) 1000)` against `(extern "abs" (Int) Int)` is `E_TYPE_MISMATCH`.

## Bad

```lisp
(extern "fabs" (Float) Float)        ; E_TYPE_MISMATCH: Float cannot cross the C boundary
(extern "abs" (a) b)                 ; E_TYPE_MISMATCH: extern types are concrete
```

## Good

```c
#include <stdint.h>
#include <string.h>
int64_t my_factorial(int64_t n) { return n <= 1 ? 1 : n * my_factorial(n - 1); }

/* float wrapper: bits in, bits out */
int64_t scale_bits(int64_t xbits) {
    double x; memcpy(&x, &xbits, 8);
    double r = x * 2.0;
    int64_t out; memcpy(&out, &r, 8);
    return out;
}
```

```lisp
(extern "my_factorial" (Int) Int)
(extern "scale_bits" (Int) Int)

(defn scale ((x Float))
  (ffi-call "zyl_float_of_bits"
    (ffi-call "scale_bits" (ffi-call "zyl_float_bits" x 1000) 1000) 1000))   ; (scale 1.5) is 3.0
```

## What C receives

| Extern type | C sees | Declare as |
|---|---|---|
| `Int`, `Bool` | the integer (0/1) | `int64_t` |
| `String` | pointer to NUL-terminated bytes | `const char *` |
| `Ptr` | raw address (from `alloc-malloc`, or returned by C) | `void *`/`char *` |
| `(Pin a)` | pointer to an 8-byte slot holding the value (`ffi-pin`) | `int64_t *` |
| `(Fn (A ...) R)` | code pointer of a top-level function (callback) | function pointer |
| an ADT or struct | pointer to `[tag][field0]...` (internal layout) | avoid depending on it |
| `Float` | rejected | pass `zyl_float_bits` as `Int` |

## Notes

- There is no `zyl_ffi.h` header; use `<stdint.h>`.
- Up to 16 arguments (`E_ARITY_MISMATCH` beyond); a count that differs from the extern is `E_TYPE_MISMATCH` ("wrong number of arguments to abs"). Arity 7+ copies stack arguments into an aligned block. The interpreter reaches foreign symbols through `zyl_ffi_timed_argv`, with the same limit and timeout enforcement.
- A closure passed as an argument is rejected (`E_INVALID_CAPABILITY`, not FFI_Pinnable); a top-level function name is passed as a code pointer.
- `zyl_*` runtime entries need no extern: they are typed by `stdlib/compiler/ffi_sigs.zyl`. See [ffi-extern-required](ffi-extern-required.md).

## See Also

- [ffi-extern-required](ffi-extern-required.md)
- [ffi-pin-passes-pointer](ffi-pin-passes-pointer.md)
- [type-value-representation](type-value-representation.md)
