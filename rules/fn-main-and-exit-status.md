# fn-main-and-exit-status

> Every executable needs `(defn main () ...)` of type `() -> Int`: no parameters, and a body that ends in an Int, which becomes the exit status. End it with an explicit `0`.

## Why It Matters

The type checker requires `main : () -> Int`. A `main` ending in `print` (which is `Unit`), a Bool or a String is `E_TYPE_MISMATCH` ("main must return the exit status, an Int, but this is `Unit`"), and a `main` with parameters is `E_TYPE_MISMATCH` too. The generated entry stub runs `main` on the big stack and returns its value as the process exit code, truncated to 8 bits by the OS (`300` exits with 44). A file with top-level `test`/`run-tests` forms gets a synthesized `main` and must **not** also define one (`E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN`). A lone file (no `zyl.pkg` beside it) with no `main` and no tests is `E_NO_MAIN`, located at its first line, from both `zyl check` and a build (before 2026-10-09 check passed it and the link failed with `PANIC: E_LINK_UNDEFINED: _ZYL_main`). A file beside a `zyl.pkg` may be a module and is not required to have one.

## Bad

Each `main` is its own error.

```lisp
(defn main () (print "hi"))       ; E_TYPE_MISMATCH: print is Unit
```

```lisp
(defn main (argc) argc)           ; E_TYPE_MISMATCH: main takes no parameters
```

## Good

```lisp
(defn report ((n Int)) (print n))
(defn count-lines ((name String)) (str-length name))

(defn main ()
  (begin
    (report (count-lines "sample.log"))   ; 10
    0))
```

## Notes

- Command-line arguments: `(ffi-call "zyl_argc" 1000)` and `(ffi-call "zyl_arg_str" i 1000)` (index 0 is the program path).
- To fail with a non-zero status, return it from `main`, call `(exit code)` (flushes output and ends the process at once, without draining actors), or `panic` (a panic exits 1).
- A `main` in a module meant to be `use`d collides with the importer's: see [pkg-library-no-main](pkg-library-no-main.md).
- `main` is never qualified to a canonical key, but the capability check still covers its body: an `ffi-call` in `main` needs `(capabilities ffi)` like any other ([pkg-lone-file-capabilities](pkg-lone-file-capabilities.md)).
- `main` is exempt from `W_UNUSED_FUNCTION` and is never inlined.

## See Also

- [test-toplevel-forms-and-run-tests](test-toplevel-forms-and-run-tests.md) - test files have no main
- [pkg-lone-file-capabilities](pkg-lone-file-capabilities.md) - what a lone file may do
