# pkg-use-what-you-construct

> Explicitly `use` every module whose types, constructors or functions your module relies on, even if they "happen to be visible".

## Why It Matters

Within one package (or a lone file and the modules next to it) the resolver splices every loaded module into one program, so a definition is visible to every module as soon as **any** module loads it. Code that leans on that works in one program and breaks in the next, the day the module that happened to load the dependency stops doing so:

- In a `match`, an unknown constructor becomes a catch-all binding, so the arms after it are reported as `E_UNREACHABLE_MATCH_ARM` — or, in the last arm, it silently catches everything.
- A call to a function nothing loaded is a located `E_UNBOUND_VARIABLE` (`call to undefined function ...`) from the type checker. Compiler source is no exception: since 2026-09-24 the compiler is built through module resolution too ([boot-module-build](boot-module-build.md)).

The prelude is the exception: `Cons`/`Nil`, `Some`/`None`, `Ok`/`Err` and the `core/core` definitions are visible everywhere without a `use`.

## Bad

```lisp
; shapes.zyl
(deftype Shape (Circle Int) (Square Int))

; area.zyl -- matches on Shape, but never uses shapes
(defn area (s)
  (match s
    (Circle r (* 3 (* r r)))       ; compiles only while some other module loads shapes;
    (Square w (* w w))))           ; otherwise E_UNREACHABLE_MATCH_ARM: `Circle` is a catch-all
```

```lisp
; app.zyl
(use area)
(use shapes)                       ; loads Shape, so area.zyl compiles here...
(defn main () (begin (print (area (Square 3))) 0))

; app2.zyl
(use area)                         ; ...and not here
(defn main () (begin (print 1) 0))
```

## Good

```lisp
; area.zyl
(use shapes)
(defn area (s)
  (match s
    (Circle r (* 3 (* r r)))
    (Square w (* w w))))
```

## Notes

- Prefer a small local type over depending on a large, wrong-direction module for one shape (the compiler's `sexp_balance.zyl` has its own `SBPair` rather than pulling in `type_system` for `Pair`).

## See Also

- [match-misspelled-last-arm](match-misspelled-last-arm.md)
- [boot-module-build](boot-module-build.md)
