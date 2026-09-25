# proj-file-io

> Use the built-in `file-open`/`file-read`/`file-write`/`file-close` forms with a literal mode and String data; check the descriptor and read with an explicit maximum size.

## Why It Matters

File handles are `Int` descriptors. `file-open` returns a negative number (-1) on failure; forgetting to check means reading from garbage. `file-read fd n` returns **up to** `n` bytes as a `String` (at most 64 MiB per read). The operands are typed:

| Form | Types |
|---|---|
| `(file-open path mode)` | `path` a `String`; `mode` a **string literal**: `"r"`, `"w"`, `"a"`, optionally with `+` or `b` (`"r+"`, `"wb"`, ...); returns `Int` |
| `(file-read fd n)` | `Int Int`, returns `String` |
| `(file-write fd data)` | `data` must be a `String`; returns the byte count (`Int`) |
| `(file-close fd)` | `Int`, returns 0 |

Any other mode, or a mode that is not a literal, is `E_TYPE_MISMATCH` (it used to compile as `"w"` and truncate the file). Writing a number needs its text first. No `file-seek`/`file-tell`/`file-size`, and `read-line` is not lowered: it type-checks as a `String` but returns a null string. In a package these need the `io` capability.

## Bad

```lisp
(file-write fd 42)                  ; E_TYPE_MISMATCH: expected `String`, found `Int`
(file-open path "x")                ; E_TYPE_MISMATCH: the mode of file-open must be a string literal ...
(file-open path mode)               ; same: the mode must be written in the call
```

## Good

```lisp
(defn slurp ((path String))
  (let fd (file-open path "r")
    (if (< fd 0)
      None
      (let content (file-read fd 1000000)
        (let _ (file-close fd)
          (Some content))))))

(defn save ((path String) (n Int))
  (let fd (file-open path "w")
    (if (< fd 0)
      false
      (let _ (file-write fd (str-concat (ffi-call "zyl_int_text" n 1000) "\n"))
        (let _ (file-close fd)
          true)))))
```

## Notes

- The runtime looks only at the mode's first letter: `r` opens read-only (so `"r+"` cannot write), `a` appends, anything else truncates and writes.
- `file-write` of a `Secret` is `E_SECRET_ESCAPE`.
- `io/io` adds `io-file-open-read`/`-write`/`-append`, `io-file-read`, `io-file-write`, `io-file-close`, `io-safe-*`, and the `OutputStream` trait (`Stdout`, `StringBuffer`).
- Relative paths resolve against the process's working directory.

## See Also

- [proj-idioms](proj-idioms.md)
- [pkg-capabilities](pkg-capabilities.md)
