# pkg-library-no-main

> Never define `main` (or other generic names an importer might define) in a module meant to be `use`d.

## Why It Matters

A `use`d file's top-level forms are spliced into the importer verbatim; `main` is never qualified or renamed. A library `main` collides with the program's own `main`, and makes every test file that uses the library fail with `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN`. `selfhost/assemble.py` strips non-driver `main`s from the compiler bundle, but that does **not** happen for a normal `use`, so compiler-stdlib files must obey this too.

## Bad

```lisp
; mathlib.zyl
(defn square (x) (* x x))
(defn main () (print (square 4)))     ; breaks every importer
```

## Good

```lisp
; mathlib.zyl
(defn square (x) (* x x))
(defn mathlib-demo () (print (square 4)))   ; a real program's main may call this
```

## Diagnosis

`E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` in a file with no `main` of its own → check every module in the transitive `use` chain.

## See Also

- [test-program-library-tests-split](test-program-library-tests-split.md)
- [boot-assemble-bundle](boot-assemble-bundle.md)
