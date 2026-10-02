# pass-conservative-failure-direction

> Make soundness checks fail closed (reject what they cannot prove), lint-like checks fail open (miss a diagnostic rather than reject valid code), and transformations fail safe (fall back to the always-correct path).

## Why It Matters

Each kind of pass has one safe direction, and a pass that picks the wrong one either lets a wrong program through or breaks valid programs and the fixed point.

| Kind | Direction | Today |
|---|---|---|
| soundness checks | fail closed | the type pass rejects every unification failure, occurs-check failure and unknown type, and a form it has no case for (`E_CANNOT_INFER: no type for form not typed`); `E_REGION_ESCAPE` fires when a Stack bytebuf or `with-region` value *may* escape; a malformed special form is `E_MALFORMED_FORM`, not a 0; an Int `/` or `%` whose divisor is not a nonzero literal is `E_PARTIAL_OPERATION` |
| lint-like checks | fail open, being closed one by one | mutability, exhaustiveness and unused checks are syntactic: a shape they do not understand is let through. Document what they miss, because users will assume full enforcement. Three have moved to fail closed: the capability check walks every expression form by name with no default arm, test bodies and impl methods included; the `Secret` check rejects a tainted argument to a parameter not annotated `Secret` (`E_SECRET_UNANNOTATED`; calls through function values and trait methods are still not covered); and a capitalized arm head no type declares is `E_UNKNOWN_CONSTRUCTOR` rather than a catch-all binder |
| transformations | fail safe | region inference over-approximates escape (field-insensitive classes; unknown runtime functions and function-value calls are heap); reuse takes a block only when every condition holds and the size header covers the record, else allocates; the inliner skips any candidate with a node it cannot copy exactly; the native backend declines any function with a node outside its set (`mb-eligible`), which the stack machine then compiles |

A transformation's fallback must be the path that is correct for every input, and the decision to leave it must be a proof, not a guess: a missed optimization costs speed, a wrong one corrupts memory.

## Bad

```lisp
; a lowering that guesses: an unknown node computes 0
(deftype Node (Const Int) (Add Node Node) (Call Int))
(defn lower (e)
  (match e
    (Const n n)
    (Add a b (+ (lower a) (lower b)))
    (_ 0)))                            ; a Call silently becomes 0
```

## Good

```lisp
; decline in an eligibility check, and give the whole function to the general path
(deftype Node (Const Int) (Add Node Node) (Call Int))
(defn ok (e)
  (match e
    (Const _ true)
    (Add a b (and (ok a) (ok b)))
    (_ false)))
(defn lower (e)
  (match e
    (Const n n)
    (Add a b (+ (lower a) (lower b)))
    (_ 0)))                            ; unreachable: only called when ok holds
(defn fast-or-general (e (general Int))
  (if (ok e) (lower e) general))
```

The compiler's own instance is the native backend: `ml-ok` and `mb-eligible` (`codegen.zyl`) decline a function with a node outside the set, and the stack machine compiles it instead.

## See Also

- [icnf-regions-are-a-rewrite](icnf-regions-are-a-rewrite.md)
- [icnf-reuse-pass](icnf-reuse-pass.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
- [type-sound-checking](type-sound-checking.md)
