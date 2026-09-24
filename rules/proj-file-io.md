# proj-file-io

> Use the built-in `file-open`/`file-read`/`file-write`/`file-close` forms; check the descriptor and read with an explicit maximum size.

## Why It Matters

File handles are `Int` descriptors. `file-open` returns a negative number on failure; forgetting to check means reading from garbage. `file-read fd n` returns **up to** `n` bytes as a string. No `file-seek`/`file-tell`/`file-size`, and `read-line` is not lowered. In a package these need the `io` capability.

## Good

```lisp
(defn process-file (path)
  (let fd (file-open path "r")
    (if (< fd 1)
      (begin
        (print "cannot open the log file")
        (empty-stats))
      (let content (file-read fd 1000000)
        (let _ (file-close fd)
          (scan-text content))))))
```

## Notes

- Modes: `"r"` read, `"a"` append, anything else writes.
- `file-write` of a `Secret` is `E_SECRET_ESCAPE`.
- `io/io` adds `io-file-open-read`/`-write`/`-append`, `io-file-read`, `io-file-write`, `io-file-close`, `io-safe-*`, and the `OutputStream` trait (`Stdout`, `StringBuffer`).
- Relative paths resolve against the process's working directory.

## See Also

- [proj-idioms](proj-idioms.md)
- [pkg-capabilities](pkg-capabilities.md)
