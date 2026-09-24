# pkg-use-what-you-construct

> Explicitly `use` every module whose types or constructors your module builds or matches on, even if they "happen to be visible".

## Why It Matters

Without the `use`, the constructor is unknown inside your module:

- In a `match`, the unknown name becomes a catch-all, so the arms after it are reported as `E_UNREACHABLE_MATCH_ARM` — or, in the last arm, it silently catches everything.
- A function from a module you did not `use` is `E_UNBOUND_VARIABLE` at the call. Compiler source is no exception: since 2026-09-24 the compiler is built through module resolution too ([boot-module-build](boot-module-build.md)), so nothing is visible without its `use`.

## Bad

```lisp
; logstats.zyl -- builds and matches Cons/Nil lists
(defn list-nth (l k)
  (match l
    (Nil 0)                         ; Nil unknown here: catch-all
    (Cons h t (if (= k 0) h (list-nth t (- k 1))))))   ; E_UNREACHABLE_MATCH_ARM
```

## Good

```lisp
(use core/list)
(defn list-nth (l k)
  (match l
    (Nil 0)
    (Cons h t (if (= k 0) h (list-nth t (- k 1))))))
```

## Notes

- Prefer a small local type over depending on a large, wrong-direction module for one shape (the compiler's `sexp_balance.zyl` has its own `SBPair` rather than pulling in `type_system` for `Pair`).

## See Also

- [match-misspelled-last-arm](match-misspelled-last-arm.md)
- [boot-module-build](boot-module-build.md)
