# match-exhaustive-or-underscore

> Cover every variant, or end with a single `_` arm; a non-exhaustive constructor match is a compile-time error.

## Why It Matters

Exhaustiveness is a compile-time **error**, not a warning. A constructor match missing a variant with no catch-all is `E_NON_EXHAUSTIVE_MATCH` (located, naming the first missing variant). A repeated arm is `E_UNREACHABLE_MATCH_ARM`. No constructor match can fall through at run time.

## Bad

```lisp
(deftype TrafficLight (Red) (Yellow) (Green))
(defn action (light)
  (match light
    (Red "stop")))
;; error[E_NON_EXHAUSTIVE_MATCH]: match over `TrafficLight` does not cover variant `Yellow`

(match c (Red 1) (Red 2) (Green 3) (Yellow 4))
;; error[E_UNREACHABLE_MATCH_ARM]: arm `Red` is already matched by an earlier arm
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

- The checker infers the scrutinee's type from the constructor names in the arms. If a name is shared by two types, it skips the match, and ICNF lowering reports a gap instead as an **unlocated** `E_MATCH_NONEXHAUSTIVE: match does not cover every variant` ([data-unique-variant-names](data-unique-variant-names.md)).
- Spelling: the constructor check prints `E_NON_EXHAUSTIVE_MATCH`; the spec's name `E_MATCH_NONEXHAUSTIVE` is what the literal-pattern check and the lowering fallback print.
- A misspelled constructor in the last arm is a catch-all and satisfies the check ([match-misspelled-last-arm](match-misspelled-last-arm.md)).
- There is no `default`/`otherwise` keyword.

## See Also

- [match-misspelled-last-arm](match-misspelled-last-arm.md)
- [match-catch-all-last](match-catch-all-last.md)
