# boot-match-arm-call-sums

> In a match arm, bind call results with `let` before combining a constant with two or more calls in one arithmetic expression (`E_MATCH_ARM_COMPLEX`); elsewhere, calls may be combined directly.

## Why It Matters

Historically a binop whose operands were two calls silently computed 0 in stage ≥ 2 binaries, and `icnf-arm-size` with multiple calls in a match arm computed 0. That miscompile is gone. Verified 2026-09-25 in both backends (native and `ZYL_MIR=0`), with and without inlining: `(+ 1 (f n m) (g n))` in a function body, `(+ (f n m) (g n))` and `(* (f n 1) (g n))` in match arms, and nested `str-concat` of `str-concat` calls all compute correctly.

What remains is the syntactic guard `ic-arm-guard` in `icnf.zyl`: an arm body whose top-level operator application has at least one literal operand and two or more call operands is rejected at lowering with a located `error[E_MATCH_ARM_COMPLEX]: match arm combines a constant with multiple calls`. Because it fires in ICNF lowering, `zyl check` (which stops after type inference and the numeric check) does not see it; only a build does. It fires for user programs and for compiler source alike. The guard is kept because nobody has re-tested removing it; old comments in `codegen.zyl` (`cg-load-reg-args`, `icnf-arm-size`) and `error_report.zyl` still ask for the let discipline everywhere, but only the arm shape is enforced.

## Bad

```lisp
(deftype T (A Int Int) (B))
(defn f (n m) (+ n m))
(defn g (n) (* n 2))
(defn h (t)
  (match t
    (A n m (+ 1 (f n m) (g n)))              ; E_MATCH_ARM_COMPLEX when built
    (B 0)))
(defn main () (begin (print (h (A 1 2))) 0))
```

## Good

```lisp
(deftype T (A Int Int) (B))
(defn f (n m) (+ n m))
(defn g (n) (* n 2))
(defn h (t)
  (match t
    (A n m (let a (f n m) (let b (g n) (+ 1 a b))))
    (B 0)))
(defn h2 (t)
  (match t
    (A n m (+ (f n m) (g n)))                ; no literal operand: accepted and correct
    (B 0)))
(defn k (n m) (+ 1 (f n m) (g n)))           ; not in an arm: accepted and correct
(defn main () (begin (print (h (A 1 2))) (print (h2 (A 1 2))) (print (k 1 2)) 0))
```

## Notes

- N-ary binops fold left (`ic-binop-fold`).
- Removing the guard means deleting `ic-arm-guard`, reseeding, and running the full suite; see [boot-fixed-point-workflow](boot-fixed-point-workflow.md).

## See Also

- [match-arm-complex](match-arm-complex.md)
- [boot-lifted-constraints](boot-lifted-constraints.md)
