# test-compiler-internals

> Test compiler passes by `use`ing the compiler modules and asserting on their results, not only with black-box compile-fail files.

## Why It Matters

`stdlib/compiler/*.zyl` are ordinary library modules. A test can parse, lower and inspect ICNF directly, and can check the exact diagnostic (location, hint text). `tests/compile-fail/*.zyl` only prove that compilation fails: the runner does not check **which** error code was raised.

## Good

```lisp
(use allocator/allocator)
(use compiler/lexer)
(use compiler/parser)
(use compiler/ast)
(use compiler/expr_inner)
(use compiler/icnf)
(use compiler/optimization)

(test "parse-nested-ast"
  (let arena (arena-create 0)
    (let prog (zyl-parse arena "(defn f (x) (* x x))")
      (assert-equal (list-length prog) 1))))

(test "constant-folding-removes-binops"
  (let arena (arena-create 1048576)
    (let fns (ic-program arena (convert-ast-list (zyl-parse arena "(defn main () (+ (* 2 3) 4))")))
      (assert-equal (count-binops-list (opt-optimize-fns fns)) 0))))   ; count-binops-list: a total walk over Icnf (book §30.7)
(run-tests)
```

## Notes

- Examples in the tree: `tests/regression/compiler.zyl` (`zyl-lex`, `zyl-parse`, `sb-check-string`), `tests/integration/selfhost-codegen.zyl`.
- Add a `tests/regression/*.zyl` for behavior and a `tests/compile-fail/*.zyl` per new error.

## See Also

- [pass-adding-a-pass](pass-adding-a-pass.md)
- [test-regression-runner](test-regression-runner.md)
