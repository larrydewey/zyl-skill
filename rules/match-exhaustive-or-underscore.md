# match-exhaustive-or-underscore

> Cover every variant, or end with a single `_` arm; a non-exhaustive constructor match is a compile-time error.

## Why It Matters

Exhaustiveness is a compile-time **error**, not a warning. A constructor match missing a variant with no catch-all is `E_NON_EXHAUSTIVE_MATCH` (located, naming the first missing variant). A repeated arm is `E_UNREACHABLE_MATCH_ARM`. No constructor match can fall through at run time.

## Bad

```lisp
(deftype TrafficLight (Red) (Yellow) (Green))
(defn action (light)
  (match light
    (Red "stop")))   ; E_NON_EXHAUSTIVE_MATCH: does not cover variant `Yellow`
```

```lisp
(deftype TrafficLight (Red) (Yellow) (Green))
(defn code (c)
  (match c (Red 1) (Red 2) (Green 3) (Yellow 4)))   ; E_UNREACHABLE_MATCH_ARM: `Red` again
```

## Good

```lisp
(deftype TrafficLight (Red) (Yellow) (Green))
(defn action (light)
  (match light
    (Red "stop")
    (Yellow "caution")
    (Green "go")))
```

## Limits of the check

- The checker infers the scrutinee's type from the constructor names in the arms. If a name is shared by two types, it skips the match, and ICNF lowering reports a gap instead as a located `E_MATCH_NONEXHAUSTIVE` ("match does not cover every variant"), which `zyl check` does not run ([data-unique-variant-names](data-unique-variant-names.md)).
- Spelling: the constructor check prints `E_NON_EXHAUSTIVE_MATCH`; the spec's name `E_MATCH_NONEXHAUSTIVE` is what the literal-pattern check and the lowering fallback print.
- A misspelled capitalized constructor in any arm is `E_UNKNOWN_CONSTRUCTOR`, with a suggestion when a declared constructor is within edit distance 2; it no longer becomes a catch-all that satisfies the check ([match-unknown-constructor](match-unknown-constructor.md)). A lowercase arm head is still a binder, and so a catch-all.
- There is no `default`/`otherwise` keyword.

## See Also

- [match-unknown-constructor](match-unknown-constructor.md)
- [match-catch-all-last](match-catch-all-last.md)
