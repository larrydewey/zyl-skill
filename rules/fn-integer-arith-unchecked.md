# fn-integer-arith-unchecked

> Guard divisors and overflow yourself: compiled integer arithmetic wraps silently and division by zero kills the process with SIGFPE.

## Why It Matters

The spec requires checked overflow (`E_OVERFLOW`) and `E_DIVISION_BY_ZERO`. Compiled code does neither: `(+ 9223372036854775807 1)` is `-9223372036854775808`, and `(/ x 0)` executes `idiv` and dies with SIGFPE. Only the REPL interpreter (`zyl eval`) reports `E_DIVISION_BY_ZERO`, so a program can "work" in the REPL and crash compiled.

## Bad

```lisp
(defn avg (total count) (/ total count))   ; SIGFPE when count is 0
```

## Good

```lisp
(defn avg (total count)
  (if (== count 0) (Err "empty") (Ok (/ total count))))
```

## Notes

- `/` truncates toward zero: `(/ -7 2)` is -3. `%` takes the sign of the dividend: `(% -7 2)` is -1, `(% -17 5)` is -2.
- `+ - *` are n-ary and fold left: `(- 10 3 2)` is 5; `(/ 20 2 2)` is 5.
- Only `-` has a one-argument form (negation). `(+ x)`, `(* x)`, `(/ x)` evaluate to **0**.
- The optimizer never folds a division by a constant zero, so it still fails at run time rather than at compile time.
- On `Secret` operands `/` and `%` are `E_CT_VIOLATION`.

## See Also

- [syn-int-literal-range](syn-int-literal-range.md) - oversized literals become 0
- [tool-eval-differential](tool-eval-differential.md) - REPL vs compiled differences
