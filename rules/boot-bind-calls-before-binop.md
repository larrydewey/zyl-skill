# boot-bind-calls-before-binop

> In compiler source, bind each call result with `let` before combining calls in one arithmetic or `str-concat` expression.

## Why It Matters

Historically a binop whose operands were two calls silently computed 0 in stage ≥ 2 binaries, and `icnf-arm-size` with multiple calls in a match arm computed 0. The general shape no longer reproduces (as of 2026-09-23, `(+ (f 3) (g 4))`, `(* (f 2 3) (f 4 5))`, two-call sums in match arms and nested `str-concat` all compute correctly), but the checked shape — a match arm combining a constant with two or more calls — is still rejected (`E_MATCH_ARM_COMPLEX`), and the compiler source keeps the old discipline because a miscompile there breaks the fixed point rather than one program. `error_report.zyl`'s header asks the same of every `str-concat`.

## Bad

```lisp
(A n m (+ 1 (f n m) (g n)))                  ; E_MATCH_ARM_COMPLEX in an arm
(str-concat (loc-string n) (err-header c))   ; avoid in compiler source
```

## Good

```lisp
(A n m (let a (f n m) (let b (g n) (+ 1 a b))))
(icnf-add2 1 (icnf-add2 (f x) (g y)))        ; or nest a two-arg helper
(let loc (loc-string n) (let hdr (err-header c) (str-concat loc hdr)))
```

## Notes

- N-ary binops fold left (`ic-binop-fold`); `ic-binop` with 3+ args once lowered to 0.

## See Also

- [match-arm-complex](match-arm-complex.md)
- [boot-lifted-constraints](boot-lifted-constraints.md)
