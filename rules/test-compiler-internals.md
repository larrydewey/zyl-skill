# test-compiler-internals

> Test compiler passes by `use`ing the compiler modules and asserting on their results, and pin every compile-fail test to its code with `; expect-error: CODE`.

## Why It Matters

`stdlib/compiler/*.zyl` are ordinary library modules. A test can parse, lower and inspect ICNF directly, and can check exact results that a black-box compile cannot show. A `tests/compile-fail/*.zyl` file proves only that compilation fails unless it carries a `; expect-error: CODE` line; with one, the runner also requires that code in the output, so a test cannot pass by failing for the wrong reason.

## Good

```lisp
(use allocator/allocator)
(use core/list)
(use compiler/lexer)
(use compiler/parser)
(use compiler/ast)
(use compiler/expr_inner)
(use compiler/icnf)
(use compiler/optimization)

(defn count-binops (e)                 ; a partial walk over Icnf (book §30.7)
  (match e
    (IBinop _ l r (+ 1 (+ (count-binops l) (count-binops r))))
    (ICall _ args (count-binops-list args))
    (IFfi _ args (count-binops-list args))
    (IPrint a (count-binops a))
    (IIf c t f (+ (count-binops c) (+ (count-binops t) (count-binops f))))
    (IWhile c b (+ (count-binops c) (count-binops b)))
    (ILet _ v b (+ (count-binops v) (count-binops b)))
    (ISeq es (count-binops-list es))
    (IFn _ _ body _ (count-binops body))
    (_ 0)))
(defn count-binops-list (xs)
  (match xs (Nil 0) (Cons h t (+ (count-binops h) (count-binops-list t)))))

(test "parse-nested-ast"
  (let arena (arena-create 0)
    (let prog (zyl-parse arena "(defn f (x) (* x x))")
      (assert-equal (list-length prog) 1))))

(test "constant-folding-removes-binops"
  (let arena (arena-create 1048576)
    (let fns (ic-program arena (convert-ast-list (zyl-parse arena "(defn main () (+ (* 2 3) 4))")))
      (assert-equal (count-binops-list (opt-optimize-fns fns)) 0))))
(run-tests)
```

```lisp
; tests/compile-fail/stack-bytebuf-return.zyl
; expect-error: E_REGION_ESCAPE

(defn make-buf () (bytebuf Stack 16))

(defn main () (begin (print (bytebuf-cap (make-buf))) 0))
```

## Notes

- Examples in the tree: `tests/regression/compiler.zyl` (`zyl-lex`, `zyl-parse`, `sb-check-string`), `tests/regression/eval-order.zyl` (evaluation order through a trace), `tests/integration/selfhost-codegen.zyl`.
- `ic-program` on unannotated source (as above) skips the type checker; for anything that depends on types, go through `compile-to-fns` (`compiler/pipeline.zyl`), which runs every phase up to codegen.
- Add a `tests/regression/*.zyl` for behavior and a `tests/compile-fail/*.zyl` (with `; expect-error:`) per new error.

## See Also

- [pass-adding-a-pass](pass-adding-a-pass.md)
- [test-regression-runner](test-regression-runner.md)
