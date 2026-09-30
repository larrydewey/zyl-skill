# match-mixed-patterns

> Literal patterns and constructor patterns cannot be mixed in the same `match` — the compiler now emits `E_MATCH_MIXED_PATTERNS`.

## Why It Matters

Before this fix, a `match` with both literal patterns (Int, Float, String, Bool, range) and constructor patterns (ADT variants) compiled silently but computed the wrong result. Literal matches lower to an `EIf` chain; constructor matches lower to `EMatch`. Mixing them fell through to whichever lowering the first arm triggered, and the other pattern kind became a silent catch-all.

## Bad

```lisp
;; n is an Int, but Red is treated as a catch-all
(defn bad (n)
  (match n
    (1 "one")
    (Red "caught everything else")
    (_ "never reached")))
```

```lisp
;; Range + constructor also mixed
(deftype Shape (Circle Int) (Rect Int Int))
(defn bad2 (s)
  (match s
    ((range 1 10) "small")
    (Circle r "never matches")
    (_ "other")))
```

## Good

Separate the match into two, or use guards inside arms:

```lisp
;; Literal match only
(defn good-literal (n)
  (match n
    (1 "one")
    (2 "two")
    (_ "other")))

;; Constructor match only
(defn good-ctor (s)
  (match s
    (Circle r (str-concat "circle " (Show.show r)))
    (Rect w h (str-concat "rect " (Show.show w)))
    (_ "unknown")))
```

## Notes

- Literal patterns: `Int`, `Float`, `String`, `Bool`, `(range lo hi)`, and OR-combinations (`1 2 3`).
- Constructor patterns: variant names from `deftype` (with field binders) and wrapped forms `((Variant ...) body)`.
- The wildcard `_` is neither — it is allowed in both kinds as the final arm.
- This check runs at parse time (phase 2), before type checking.

## See Also

- [match-literal-requires-underscore](match-literal-requires-underscore.md)
- [match-arm-complex](match-arm-complex.md)
- [syn-list-literals-and-quote](syn-list-literals-and-quote.md)