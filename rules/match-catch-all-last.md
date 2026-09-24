# match-catch-all-last

> Put `_` last, exactly once; an arm after a catch-all is `E_UNREACHABLE_MATCH_ARM`.

## Why It Matters

Arms are tried in source order and the first match wins; there is no fallthrough. A catch-all followed by more arms is rejected, which is also the mechanism that catches a misspelled constructor in a non-last arm.

## Bad

```lisp
(match x
  (_ 0)
  (Some v v))        ; E_UNREACHABLE_MATCH_ARM
```

## Good

```lisp
(match x
  (Some v v)
  (_ 0))
```

## Notes

- A guard on the trailing `_` arm is ignored.
- `_` inside an arm (field discards) may repeat freely; the rule is about the arm head.

## See Also

- [match-misspelled-last-arm](match-misspelled-last-arm.md)
