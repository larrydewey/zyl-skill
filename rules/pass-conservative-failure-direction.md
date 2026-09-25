# pass-conservative-failure-direction

> Make soundness checks fail closed (reject what they cannot prove), lint-like checks fail open (miss a diagnostic rather than reject valid code), and transformations fail safe (fall back to the always-correct path).

## Why It Matters

Each kind of pass has one safe direction, and a pass that picks the wrong one either lets a wrong program through or breaks valid programs and the fixed point.

| Kind | Direction | Today |
|---|---|---|
| soundness checks | fail closed | the type pass rejects every unification failure, occurs-check failure and unknown type, and a form it has no case for (`E_CANNOT_INFER: no type for form not typed`); `E_REGION_ESCAPE` fires when a Stack bytebuf or `with-region` value *may* escape; a malformed special form is `E_MALFORMED_FORM`, not a 0 |
| lint-like checks | fail open | mutability, capability, `Secret`, exhaustiveness and unused checks are syntactic: a shape they do not understand is let through. Document what they miss, because users will assume full enforcement |
| transformations | fail safe | region inference over-approximates escape (field-insensitive classes; unknown runtime functions and function-value calls are heap); reuse takes a block only when every condition holds and the size header covers the record, else allocates; the inliner skips any candidate with a node it cannot copy exactly; the native backend declines any function with a node outside its set (`mb-eligible`), which the stack machine then compiles |

A transformation's fallback must be the path that is correct for every input, and the decision to leave it must be a proof, not a guess: a missed optimization costs speed, a wrong one corrupts memory.

## Bad

```lisp
; a native-backend lowering case that guesses: an unknown node computes 0
(_ (ml-const ms 0))                    ; safe only because ml-ok already declined the function
```

## Good

```lisp
; decline in the eligibility check; the stack machine handles the whole function
(defn ml-ok (st env e)
  (match e
    ...
    (_ false)))
```

## See Also

- [icnf-regions-are-a-rewrite](icnf-regions-are-a-rewrite.md)
- [icnf-reuse-pass](icnf-reuse-pass.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
- [type-sound-checking](type-sound-checking.md)
