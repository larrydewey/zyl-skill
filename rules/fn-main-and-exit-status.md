# fn-main-and-exit-status

> Every executable needs `(defn main () ...)` with no parameters; its value is the exit status, so end it with an explicit `0`.

## Why It Matters

The generated entry stub runs your `main` on the big stack and returns its value as the process exit code. A `main` ending in an arbitrary expression exits with that value (truncated to 8 bits by the OS). A file with top-level `test`/`run-tests` forms gets a synthesized `main` and must **not** also define one (`E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN`).

## Bad

```lisp
(defn main ()
  (process-file "in.txt"))      ; exit status = whatever process-file returns
```

## Good

```lisp
(defn main ()
  (begin
    (report (process-file "sample.log"))
    0))
```

## Notes

- `print` evaluates to 0, so a `main` ending in `print` exits 0.
- Command-line arguments: the runtime keeps argc/argv (`zyl_argc`, `zyl_arg_str` via `ffi-call`).
- A `main` in a module meant to be `use`d collides with the importer's: see [pkg-library-no-main](pkg-library-no-main.md).
- `main` is never qualified to a canonical key, which also means the package capability check skips its body.

## See Also

- [test-toplevel-forms-and-run-tests](test-toplevel-forms-and-run-tests.md) - test files have no main
- [pkg-capabilities](pkg-capabilities.md) - `main` escapes capability checking
