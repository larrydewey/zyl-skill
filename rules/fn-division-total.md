# fn-division-total

> Divide by a nonzero integer literal, or say what a zero divisor means: `div!`/`rem!` stop the program with `E_DIVISION_BY_ZERO`, `div?`/`rem?` return an `(Option Int)`. Any other `/` or `%` on Int is `E_PARTIAL_OPERATION`.

## Why It Matters

Spec §20.3: Int division is total only when the source vouches for its divisor. `/` and `%` accept a divisor that is a nonzero integer literal, such as `(/ n 8)` or `(% i 60)`. A variable, a call or a literal `0` is refused at that operand. No guard is recognised: an `if` that tests the divisor first does not make `/` total, because the check is syntactic and the author is expected to write the spelled form instead.

| Form | Zero divisor | Result |
|---|---|---|
| `(/ a 8)`, `(% a 8)` | impossible, the literal is nonzero | `Int` |
| `(div! a b)`, `(rem! a b)` | the program stops: `PANIC: error[E_DIVISION_BY_ZERO]: division by zero in `div!``, catchable with `try` | `Int` |
| `(div? a b)`, `(rem? a b)` | `None` | `(Option Int)` |

No arithmetic reaches the processor's divide fault any more: every `idiv` with a non-literal divisor is preceded by a zero test and an `INT_MIN / -1` test, so the old SIGFPE crash (exit 136, uncatchable, buffered output lost) is gone.

## Bad

```lisp
(defn average ((total Int) (count Int))
  (if (= count 0) 0 (/ total count)))   ; E_PARTIAL_OPERATION: the guard does not count
```

## Good

```lisp
(defn average ((total Int) (count Int))
  (match (div? total count)
    (Some q q)
    (None 0)))
(defn bucket ((h Int)) (% h 64))
(defn main ()
  (begin
    (print (average 10 4))     ; 2
    (print (average 10 0))     ; 0
    (print (bucket -7))        ; -7
    (print (div! -7 2))        ; -3
    (print (rem! -17 5))       ; -2
    (print (div! 10 0))        ; PANIC: error[E_DIVISION_BY_ZERO]: division by zero in `div!`
    0))
```

## Notes

- Quotients truncate toward zero and a remainder takes the dividend's sign, as `idiv` does: `(div! -7 2)` is -3, `(rem! -7 2)` is -1.
- `INT_MIN` divided by `-1` has no Int result: it is `E_OVERFLOW` from `/`, `div!` and `div?` alike, whatever the package's numeric policy. `(rem! INT_MIN -1)` is 0.
- Division needs no `(numeric ...)` declaration; only `+ - *` do ([fn-numeric-policy](fn-numeric-policy.md)).
- `div?` and `rem?` evaluate both operands once, left to right. They are rewritten on the parse tree into a `let`/`if` around `div!`/`rem!`.
- All four spelled forms take Int operands only. Float `/` by zero is infinity or NaN, as IEEE-754 says.
- A division by a constant other than 0 and -1 compiles to a multiply by a magic number or a shift, never to `idiv`.
- On `Secret` operands `/` and `%` are `E_CT_VIOLATION`. Known gap (2026-10-02): the secret checker does not yet flag `div!`, `rem!`, `div?` or `rem?` on a Secret, so never divide key material with them either.
- `+ - *` are n-ary and fold left: `(- 10 3 2)` is 5. `(/ x)` and `(% x)` with one operand are `E_ARITY_MISMATCH`, from a build and from `zyl check` (the arity is checked while lowering to ICNF, which `check` now runs).

## See Also

- [fn-numeric-policy](fn-numeric-policy.md)
- [err-partial-ops-spelled](err-partial-ops-spelled.md) - the same `!`/`?` spelling for library functions
- [err-try-catches-panic-not-err](err-try-catches-panic-not-err.md) - what `try` catches
