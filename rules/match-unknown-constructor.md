# match-unknown-constructor

> Spell every constructor in a match arm exactly: a capitalized arm head that no type declares is `E_UNKNOWN_CONSTRUCTOR`, with the nearest real constructor suggested. Write a catch-all binder in lowercase, or as `_`.

## Why It Matters

An arm head that is not a known constructor used to be a silent catch-all, so a misspelled constructor in the last arm matched everything the earlier arms did not. Since 2026-10-02 the compiler refuses it: an arm head spelled like a constructor (capitalized) that no `deftype` in scope declares is `E_UNKNOWN_CONSTRUCTOR`, located at the head, suggesting a declared constructor within edit distance 2. The trap is gone wherever the arm is.

The same error appears when a constructor exists but is not visible: a module that matches on another module's variants must `use` the module that declares them.

## Bad

```lisp
(deftype Light (Red) (Yellow) (Green))
(defn action (l)
  (match l
    (Red "stop")
    (Yellow "caution")
    (Gren "go")))   ; E_UNKNOWN_CONSTRUCTOR: write `Green` if you meant that constructor
```

## Good

```lisp
(deftype Light (Red) (Yellow) (Green))
(defn action (l)
  (match l
    (Red "stop")
    (Yellow "caution")
    (Green "go")))
(defn describe (l)
  (match l
    (Red "red")
    (other "not red")))   ; lowercase: a deliberate binder
(defn main () (begin (print (action Green)) (print (describe Green)) 0))
```

## Notes

- A lowercase arm head is a binder and matches everything; `_` is the discard. Any arm after either is `E_UNREACHABLE_MATCH_ARM`.
- With no constructor close enough, the help says to write a constructor some `deftype` declares.
- List every variant explicitly when practical, so a variant added later fails exhaustiveness (`E_NON_EXHAUSTIVE_MATCH`) instead of silently reaching `_`.

## See Also

- [match-exhaustive-or-underscore](match-exhaustive-or-underscore.md)
- [match-catch-all-last](match-catch-all-last.md)
- [pkg-use-what-you-construct](pkg-use-what-you-construct.md)
- [data-unique-variant-names](data-unique-variant-names.md)
