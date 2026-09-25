# fn-main-and-exit-status

> Every executable needs `(defn main () ...)` of type `() -> Int`: no parameters, and a body that ends in an Int, which becomes the exit status. End it with an explicit `0`.

## Why It Matters

The type checker requires `main : () -> Int`. A `main` ending in `print` (which is `Unit`), a Bool or a String is `E_TYPE_MISMATCH` ("cannot unify Unit with Int"), and a `main` with parameters is `E_TYPE_MISMATCH` too. The generated entry stub runs `main` on the big stack and returns its value as the process exit code, truncated to 8 bits by the OS (`300` exits with 44). A file with top-level `test`/`run-tests` forms gets a synthesized `main` and must **not** also define one (`E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN`). A file with no `main` and no tests fails at link time (`undefined reference to _ZYL_main`).

## Bad

```lisp
(defn main () (print "hi"))       ; E_TYPE_MISMATCH: print is Unit

(defn main (argc) 0)              ; E_TYPE_MISMATCH: main takes no parameters

(defn main ()
  (process-file "in.txt"))        ; compiles if it returns an Int: exit status = that value
```

## Good

```lisp
(defn main ()
  (begin
    (report (process-file "sample.log"))
    0))
```

## Notes

- Command-line arguments: `(ffi-call "zyl_argc" 1000)` and `(ffi-call "zyl_arg_str" i 1000)` (index 0 is the program path).
- To fail with a non-zero status, return it from `main`, call `(exit code)` (flushes output and ends the process at once, without draining actors), or `error` (a panic exits 1).
- A `main` in a module meant to be `use`d collides with the importer's: see [pkg-library-no-main](pkg-library-no-main.md).
- `main` is never qualified to a canonical key, which also means the package capability check skips its body.
- `main` is exempt from `W_UNUSED_FUNCTION` and is never inlined.

## See Also

- [test-toplevel-forms-and-run-tests](test-toplevel-forms-and-run-tests.md) - test files have no main
- [pkg-capabilities](pkg-capabilities.md) - `main` escapes capability checking
