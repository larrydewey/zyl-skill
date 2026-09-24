# match-literal-requires-underscore

> Literal, OR and range matches must end with `_`, bind nothing, and cannot be mixed with constructor arms.

## Why It Matters

A match whose arms start with literals is an implementation extension compiled to an `if` chain. Without a trailing `_` it is `E_MATCH_NONEXHAUSTIVE` (raised at parse time). Literal patterns bind no names: refer to the value through the scrutinee's own variable. Mixing literal and constructor arms in one `match` is **not diagnosed** and compiles through two incompatible mechanisms.

## Bad

```lisp
(match code (200 "OK") (404 "Not Found"))
;; E_MATCH_NONEXHAUSTIVE: a literal-pattern match must end with a `_` arm

(match v (0 "zero") (Some x "some"))   ; mixed: undiagnosed nonsense
```

## Good

```lisp
(defn status-text (code)
  (match code
    (200 "OK")
    (301 302 "Redirect")              ; OR-pattern
    ((range 500 599) "Server Error")  ; inclusive both ends
    (_ "Other")))

(defn command (s)
  (match s
    ("start" 1)                        ; strings compare by content
    ("stop" 2)
    (_ 0)))
```

## Notes

- Literals may be integers, floats, strings or booleans.
- Each arm's test is its alternatives joined by "or", then "and" the guard.

## See Also

- [match-guards-literal-arms-only](match-guards-literal-arms-only.md)
- [fn-conditionals](fn-conditionals.md)
