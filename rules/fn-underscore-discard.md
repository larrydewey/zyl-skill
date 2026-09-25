# fn-underscore-discard

> Use `_` (or a `_`-prefixed name) for anything deliberately unused; never invent dummy names.

## Why It Matters

`_` is the one discard: in patterns, parameter lists and `let`. It may repeat (`(defn f (_ _) 1)` is legal; `(Triangle _ _ _ 0)` too). `_` and any `_`-prefixed name are exempt from `W_UNUSED_*`, `W_SHADOWED_BINDING` and `E_DUPLICATE_PARAMETER`. The old `d1`/`d2` dummy names were removed from the compiler tree and must not return.

## Bad

```lisp
(defn first-of (a d1) a)          ; W_UNUSED_PARAMETER, and noise
(match xs (Cons h t h) (Nil 0))   ; no warning, but `t` reads as if it mattered
```

## Good

```lisp
(defn first-of (a _) a)
(match xs (Cons h _ h) (Nil 0))
(let _ (file-close fd) content)   ; run for effect
(defn handler (_msg) 0)           ; named but exempt
```

## Notes

- Warnings (`W_UNUSED_FUNCTION`, `W_UNUSED_PARAMETER`, `W_UNUSED_VARIABLE`, `W_SHADOWED_BINDING`) go to stderr and never stop a build. `main` is exempt from `W_UNUSED_FUNCTION`.
- Match-arm pattern binders are checked for shadowing but not for being unused, so an unused `t` above is silent; write `_` anyway.
- The language server publishes these warnings (the unused check's `W_UNUSED_*` and `W_SHADOWED_BINDING`) as Warning diagnostics.
- Hygiene-renamed macro binders keep their leading underscore.

## See Also

- [match-arm-shape](match-arm-shape.md) - one binder per field
- [syn-naming-conventions](syn-naming-conventions.md)
