# match-literal-requires-underscore

> Literal, OR and range matches must end with `_`, bind nothing, and cannot be mixed with constructor arms.

## Why It Matters

A match whose arms start with literals is an implementation extension compiled to an `if` chain. Without a trailing `_` it is `E_MATCH_NONEXHAUSTIVE` ("a literal-pattern match must end with a `_` arm", raised at parse time). Literal patterns bind no names: refer to the value through the scrutinee's own variable. Mixing literal and constructor arms in one `match` is now rejected at parse time with `E_MATCH_MIXED_PATTERNS` (see [match-mixed-patterns](match-mixed-patterns.md)).

## Bad

```lisp
(defn text (code)
  (match code (200 "OK") (404 "Not Found")))   ; E_MATCH_NONEXHAUSTIVE: must end with `_`
```

```lisp
(defn kind (v)
  (match v (0 "zero") (Some _ "some") (_ "o")))  ; E_MATCH_MIXED_PATTERNS
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

(defn main ()
  (begin
    (print (status-text 302))                 ; Redirect
    (print (status-text 503))                 ; Server Error
    (print (command (str-concat "st" "op")))  ; 2
    0))
```

## Notes

- Literals may be integers, floats, strings or booleans; all arms' literals and the scrutinee have one type.
- Each arm's test is its alternatives joined by "or", then "and" the guard.

## See Also

- [match-guards-literal-arms-only](match-guards-literal-arms-only.md)
- [fn-conditionals](fn-conditionals.md)
