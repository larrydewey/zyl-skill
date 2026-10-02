# match-catch-all-last

> Put `_` last, exactly once, with no guard; an arm after a catch-all is `E_UNREACHABLE_MATCH_ARM`.

## Why It Matters

Arms are tried in source order and the first match wins; there is no fallthrough. A catch-all followed by more arms is rejected ("the arm `_` matches every value, so the arms after it never run"). A repeated constructor arm is rejected the same way ("arm `Red` is already matched by an earlier arm and can never run"). A misspelled capitalized constructor is no longer a catch-all in any position: it is `E_UNKNOWN_CONSTRUCTOR` ([match-unknown-constructor](match-unknown-constructor.md)).

## Bad

Each definition is its own error.

```lisp
(defn f (x)
  (match x
    (_ 0)
    (Some v v)))                           ; E_UNREACHABLE_MATCH_ARM

(defn g (n v) (match n (1 2) (_ (when v) 5)))      ; E_MATCH_NONEXHAUSTIVE: a guarded `_` is no catch-all
(defn h (o v) (match o (Some x x) (_ (when v) 0))) ; E_NESTED_PATTERN
```

## Good

```lisp
(defn f (x)
  (match x
    (Some v v)
    (_ 0)))
```

## Notes

- A guard on the trailing `_` arm is an error, not ignored: in a literal match it leaves the match without a catch-all, and in a constructor match the guard reads as a nested pattern.
- `_` inside an arm (field discards) may repeat freely; the rule is about the arm head.

## See Also

- [match-unknown-constructor](match-unknown-constructor.md)
