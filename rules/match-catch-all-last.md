# match-catch-all-last

> Put `_` last, exactly once, with no guard; an arm after a catch-all is `E_UNREACHABLE_MATCH_ARM`.

## Why It Matters

Arms are tried in source order and the first match wins; there is no fallthrough. A catch-all followed by more arms is rejected (`catch-all arm `_` is followed by more arms that can never run`), which is also the mechanism that catches a misspelled constructor in a non-last arm. A repeated constructor arm is rejected the same way (`arm `Red` is already matched by an earlier arm`).

## Bad

```lisp
(match x
  (_ 0)
  (Some v v))        ; E_UNREACHABLE_MATCH_ARM

(match n (1 2) (_ (when v) 5))           ; E_MATCH_NONEXHAUSTIVE: the guarded `_` is not a catch-all
(match o (Some x x) (_ (when v) 0))      ; E_NESTED_PATTERN
```

## Good

```lisp
(match x
  (Some v v)
  (_ 0))
```

## Notes

- A guard on the trailing `_` arm is an error, not ignored: in a literal match it leaves the match without a catch-all, and in a constructor match the guard reads as a nested pattern.
- `_` inside an arm (field discards) may repeat freely; the rule is about the arm head.

## See Also

- [match-misspelled-last-arm](match-misspelled-last-arm.md)
