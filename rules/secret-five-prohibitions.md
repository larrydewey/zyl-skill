# secret-five-prohibitions

> Never branch on, index with, divide by, print, or send a `Secret`, and pass it to C only through `ffi-pin`.

## Why It Matters

Each prohibited use leaks through timing, cache or an output channel. They are compile-time errors:

| A secret may not… | Error |
|---|---|
| steer control flow: `if`/`while`/`for`/`cond` condition, `match` subject | `E_CT_VIOLATION` |
| address memory: index of `w-get`, `w-set`, `list-nth`, `alloc-read-int`, byte load/store offsets | `E_CT_VIOLATION` |
| be an operand of `/` or `mod`/`%` | `E_CT_VIOLATION` |
| reach `print` | `E_SECRET_DEBUG` |
| reach `spawn`, `send`, `file-write` | `E_SECRET_ESCAPE` |
| be a raw `ffi-call` argument | `E_FFI_PIN_REQUIRED` |
| be consumed into a public result without `zeroize` | `E_ZEROIZE_MISSING` (warning) |

## Bad

```lisp
(defn leak ((k Secret))
  (if (= k 0) 1 2))
;; PANIC: in `local/main@0::leak::leak`: E_CT_VIOLATION: secret-dependent branch ...
```

## Good

```lisp
(use math/secret/secret)
(defn pick ((flag Secret) a b)
  (ct-select flag a b))             ; mask, no branch
```

## Notes

- Allowed on secrets: `+ - *`, `bit-and bit-or bit-xor bit-not`, shifts — fixed, branchless instruction sequences.
- Loop counts must come from public values (lengths, limb counts), never a secret's value.
- `Secret` is deliberately not `Send`.

## See Also

- [secret-branchless-masks](secret-branchless-masks.md)
- [secret-declassify-explicitly](secret-declassify-explicitly.md)
