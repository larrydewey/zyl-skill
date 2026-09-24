# match-exhaustive-or-underscore

> Cover every variant, or end with a single `_` arm; a match that falls through evaluates to 0.

## Why It Matters

Exhaustiveness is a compile-time **error**, not a warning. A constructor match missing a variant with no catch-all is `E_NON_EXHAUSTIVE_MATCH` (located, naming the first missing variant). It matters for safety: the generated code tests tags in order and yields 0 if nothing matches.

## Bad

```lisp
(deftype TrafficLight (Red) (Yellow) (Green))
(defn action (light)
  (match light
    (Red "stop")))
;; error[E_NON_EXHAUSTIVE_MATCH]: match over `TrafficLight` does not cover variant `Yellow`
```

## Good

```lisp
(defn action (light)
  (match light
    (Red "stop")
    (Yellow "caution")
    (Green "go")))
```

## Limits of the check

- The scrutinee's type is inferred from the constructor names in the arms, not from type inference. If a name is shared by two types, the match is **skipped** entirely.
- Repeated arms are not reported: `(match c (Red 1) (Red 2) (Green 3) (Yellow 4))` compiles; the second `Red` is dead.
- Spelling: the constructor check prints `E_NON_EXHAUSTIVE_MATCH`; the spec's name `E_MATCH_NONEXHAUSTIVE` is what the literal-pattern check prints.
- There is no `default`/`otherwise` keyword.

## See Also

- [match-misspelled-last-arm](match-misspelled-last-arm.md)
- [match-catch-all-last](match-catch-all-last.md)
