# det-left-to-right

> Rely on strict left-to-right evaluation everywhere, and never write code (or compiler passes) that reorders side effects.

## Why It Matters

Determinism is non-negotiable (spec P1, P5, §27). Evaluation order is fixed: in `(f a b c)`, evaluate `f`, then `a`, `b`, `c`, then apply. This holds for function and constructor arguments, arithmetic and comparison operands, `let` (initializer before body), `if` (condition, then exactly one branch), `begin` (in order, value of the last), `match` (scrutinee once, arms in source order). Code generation stages every argument into its own scratch slot before loading registers, so one argument's evaluation cannot clobber another.

## Good

```lisp
(defn noisy ((label String) v) (begin (print label) v))
(print (+ (noisy "first" 1) (noisy "second" 2)))
;; first
;; second
;; 3
```

## Notes

- Optimizations are safe-only: integer constant folding and dead-branch elimination; nothing is reordered and nothing with a side effect is ever folded away.
- Macro hygiene counters, lambda-lift names (`zyl_fresh_id`) and struct tags are all assigned in source order.

## See Also

- [det-ordered-collections](det-ordered-collections.md)
- [det-nondeterminism-sources](det-nondeterminism-sources.md)
