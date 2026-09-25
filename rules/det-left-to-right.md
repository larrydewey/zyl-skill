# det-left-to-right

> Rely on strict left-to-right evaluation everywhere, and never write code (or compiler passes) that reorders side effects.

## Why It Matters

Determinism is non-negotiable (spec P1, P5, §27). Evaluation order is fixed: in `(f a b c)`, evaluate `f`, then `a`, `b`, `c`, then apply. This holds for function and constructor arguments (including `make-T`, list literals `[a b c]` and quasiquote), arithmetic and comparison operands, byte operations (buffer, then offset, then value), `let` (initializer before body), `if` (condition, then exactly one branch), `begin` (in order, value of the last), `match` (scrutinee once, arms in source order). A `set!` inside a later operand never changes the value an earlier operand already produced, and a call through a local function value reads the function before its arguments run.

## Good

```lisp
(defn noisy ((label String) v) (begin (print label) v))
(defn main ()
  (begin
    (print (+ (noisy "first" 1) (noisy "second" 2)))
    0))
;; first
;; second
;; 3

(let-mut x 1
  (+ x (begin (set! x 5) 1)))    ; 2: x was read before the set!
```

## Notes

- History: until 2026-09-25 (`4892ede`) the compiler got this wrong in two places: byte loads and stores evaluated the offset before the buffer (the runtime takes them in the other order), and a call through a local function value read the local after the arguments (so `(f (begin (set! f g) 1))` called `g`). `tests/regression/eval-order.zyl` now pins operators, calls, constructors, list literals, quasiquote, `set!` in later operands, callee-first and byte operations with a trace.
- Codegen fast paths load a constant or local operand straight into its register, and when only the left operand is one, evaluate the right operand first; they apply only when the other operand contains no `set!`, since loading a constant or an unchanged local has no effect and so nothing observable moves.
- Optimizations never reorder effects: constant folding and dead-branch elimination; inlining binds the arguments first, in order, by nested `let`s; copy propagation replaces a `let` of a local that is never `set!`; the reuse pass writes a new value into a dead one's block without changing what the program computes.
- Macro hygiene counters, lambda-lift names (`zyl_fresh_id`) and struct tags are all assigned in source order.

## See Also

- [det-ordered-collections](det-ordered-collections.md)
- [det-nondeterminism-sources](det-nondeterminism-sources.md)
