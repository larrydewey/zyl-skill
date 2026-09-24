# match-misspelled-last-arm

> Spell every constructor in a match arm exactly: an arm head that is not a known constructor is a catch-all that binds nothing.

## Why It Matters

The compiler treats *any* identifier in arm-head position that is not a known constructor as a catch-all, with or without field binders. Anywhere but the last arm this is caught (`E_UNREACHABLE_MATCH_ARM`, because arms follow a catch-all). **In the last arm it silently matches everything the earlier arms did not**, and its "field binders" bind nothing. The same happens when a constructor is unknown inside a module because the module did not `use` the file defining it.

## Bad

```lisp
(deftype Light (Red) (Yellow) (Green))
(defn action (l)
  (match l
    (Red "stop")
    (Yellow "caution")
    (Gren "go")))       ; typo: catch-all. Exhaustiveness satisfied. No error.
```

## Good

```lisp
(defn action (l)
  (match l
    (Red "stop")
    (Yellow "caution")
    (Green "go")))

(defn is-red (l)
  (match l
    (Red true)
    (_ false)))        ; deliberate wildcard: always `_`
```

## Notes

- Use `_` for every intentional wildcard, so any other catch-all is suspicious in review.
- List every variant explicitly when practical: then a typo in a non-last arm is caught, and a new variant added later fails exhaustiveness instead of silently hitting `_`.
- Symptom in modules: `E_UNREACHABLE_MATCH_ARM` on the arm after `Nil` means `Nil` is unknown there — add `(use core/list)`.

## See Also

- [match-exhaustive-or-underscore](match-exhaustive-or-underscore.md)
- [pkg-use-what-you-construct](pkg-use-what-you-construct.md)
- [data-unique-variant-names](data-unique-variant-names.md)
