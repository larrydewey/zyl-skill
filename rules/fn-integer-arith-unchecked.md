# fn-integer-arith-unchecked

> Guard divisors and overflow yourself: compiled integer arithmetic wraps silently, and division by zero kills the process with SIGFPE, which `try` cannot catch and which discards buffered output.

## Why It Matters

The spec requires checked overflow (`E_OVERFLOW`) and `E_DIVISION_BY_ZERO`. Compiled code does neither: `(+ 9223372036854775807 1)` is `-9223372036854775808`, and a division by a zero divisor executes `idiv` and dies with SIGFPE (exit status 136 from a shell). That is a signal, not a panic: `try`/`catch` and `recover` do not see it, and anything printed earlier that was still in the stdout buffer is lost, so the program can appear to die before its first line of output. Only the REPL interpreter (`zyl eval`, `zyl repl`) reports `E_DIVISION_BY_ZERO`, so a program can "work" in the REPL and crash compiled.

## Bad

```lisp
(defn avg (total count) (/ total count))   ; SIGFPE when count is 0

(defn main ()
  (begin
    (print "start")                        ; never appears: lost with the buffer
    (print (try (avg 10 0) (catch _ 99)))  ; try does not catch SIGFPE
    0))
```

## Good

```lisp
(defn avg (total count)
  (if (== count 0) (Err "empty") (Ok (/ total count))))
```

## Notes

- `/` truncates toward zero: `(/ -7 2)` is -3. `%` takes the sign of the dividend: `(% -7 2)` is -1, `(% -17 5)` is -2.
- Division and remainder by a **constant** divisor no longer use `idiv` (2026-09-25): a power of two becomes a biased shift, 1 a move, and any other constant a multiply by a magic number (`zyl_div_magic`/`zyl_div_shift`). The results are exactly `idiv`'s, including negative dividends and `INT64_MIN`/`INT64_MAX`. The constants `0` and `-1` keep `idiv`, so `(/ x 0)` still raises SIGFPE and `(/ INT64_MIN -1)` still traps; the optimizer never folds a division by a constant zero into a compile-time value.
- `+ - *` are n-ary and fold left: `(- 10 3 2)` is 5; `(/ 20 2 2)` is 5.
- One operand: `(- x)` negates, `(+ x)` and `(* x)` are `x`. `(/ x)`, `(% x)` and zero-operand arithmetic are `E_ARITY_MISMATCH` (an unlocated "operator N needs two operands").
- Operands must both be Int or both Float ([fn-int-float-separation](fn-int-float-separation.md)).
- On `Secret` operands `/` and `%` are `E_CT_VIOLATION`.

## See Also

- [syn-int-literal-range](syn-int-literal-range.md) - oversized literals become 0
- [tool-eval-differential](tool-eval-differential.md) - REPL vs compiled differences
- [err-try-catches-error-not-err](err-try-catches-error-not-err.md) - what `try` does catch
