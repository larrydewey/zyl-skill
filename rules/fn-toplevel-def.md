# fn-toplevel-def

> Use a top-level `(def name expr)` for constants: an immutable global, evaluated once in source order before `main` or the tests.

## Why It Matters

Until 2026-09-24 a top-level `def` was unreadable in compiled code (`E_UNBOUND_VARIABLE`), so older code uses zero-argument `defn`s for constants. Both work now. A `def` is computed once (its side effects run once, before `main`), while a zero-argument `defn` recomputes its body on every call.

## Bad

```lisp
(def limit 10)
(defn main () (begin (set! limit 20) 0))   ; E_MUT_CONFLICT: defs are immutable
```

## Good

```lisp
(def max-size 1000)
(def app-name "MyApp")
(pub def version "1.0")                     ; exported from its module
(defn main ()
  (begin
    (print max-size)
    (print app-name)
    0))
```

## Notes

- Types are inferred as for any expression: String, Float and ADT defs print correctly.
- A def is **not generalized** (the value restriction): `(def empty Nil)` has one element type, so using it as both a `(List Int)` and a `(List String)` is `E_TYPE_MISMATCH`. Use a zero-argument `defn` for a polymorphic constant.
- A def may use functions and earlier or later defs; each is computed on first need, and every def is forced in source order before `main`'s body.
- A local `let` of the same name shadows the def.
- Implementation: `convert-program` in `expr_inner.zyl` turns each def into a caching getter (`zyl_global_*` runtime cells) and each use into a call.
- There is still no global *mutable* state.

## See Also

- [own-regions-status](own-regions-status.md) - the Global region
- [tool-repl](tool-repl.md) - REPL `def` semantics
