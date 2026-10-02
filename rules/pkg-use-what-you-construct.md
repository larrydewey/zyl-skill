# pkg-use-what-you-construct

> Explicitly `use` every module whose types, constructors or functions your module relies on, even if they "happen to be visible".

## Why It Matters

Within one package (or a lone file and the modules next to it) the resolver splices every loaded module into one program, so a definition is visible to every module as soon as **any** module loads it. Code that leans on that works in one program and breaks in the next, the day the module that happened to load the dependency stops doing so:

- In a `match`, a capitalized arm head that no loaded type declares is `E_UNKNOWN_CONSTRUCTOR` ([match-unknown-constructor](match-unknown-constructor.md)). Before 2026-10-02 it was a silent catch-all binding.
- A call to a function nothing loaded is a located `E_UNBOUND_VARIABLE` (``` `f` is not defined ```, with a did-you-mean when a near name is visible) from the type checker. Compiler source is no exception: since 2026-09-24 the compiler is built through module resolution too ([boot-module-build](boot-module-build.md)).

The prelude is the exception: `Cons`/`Nil`, `Some`/`None`, `Ok`/`Err` and the `core/core` definitions are visible everywhere without a `use`.

## Bad

```lisp
; shapes.zyl
(deftype Shape (Circle Int) (Square Int))

; area.zyl -- matches on Shape, but never uses shapes
(defn area (s)
  (match s
    (Circle r (* 3 (* r r)))       ; compiles only while some other module loads shapes;
    (Square w (* w w))))           ; otherwise E_UNKNOWN_CONSTRUCTOR: no type declares `Circle`
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

Built in one directory, `app.zyl` prints 9 and `app2.zyl` fails with `E_UNKNOWN_CONSTRUCTOR` in `area.zyl` (verified 2026-10-02).

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

- A `(numeric ...)` or `(capabilities ...)` form belongs in the compiled file (or `zyl.pkg`), never in a `use`d module: there it is `E_MALFORMED_FORM` ("the policy belongs to the whole package").
- Prefer a small local type over depending on a large, wrong-direction module for one shape (the compiler's `sexp_balance.zyl` has its own `SBPair` rather than pulling in `type_system` for `Pair`).

## See Also

- [match-unknown-constructor](match-unknown-constructor.md)
- [boot-module-build](boot-module-build.md)
