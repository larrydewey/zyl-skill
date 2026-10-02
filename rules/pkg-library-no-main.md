# pkg-library-no-main

> Never define `main` (or other generic names an importer might define) in a module meant to be `use`d.

## Why It Matters

`main` is the one name that is never qualified to its module. A library `main` collides with the program's own: the build fails with `E_DUPLICATE_DEFINITION` pointing at the program's `main` (verified 2026-09-24), and a test file that uses the library fails with `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN`. Compiler-stdlib files obey this too: the compiler itself is built by `use`ing them from `selfhost/driver.zyl`.

## Bad

```lisp
; mathlib.zyl
(defn sq (x) (* x x))
(defn main () (begin (print (sq 4)) 0))   ; breaks every importer
```

## Good

```lisp
; mathlib.zyl
(defn sq (x) (* x x))
(defn mathlib-demo () (print (sq 4)))   ; a real program's main may call this
```

Verified 2026-10-02: a program that `use`s the Bad `mathlib.zyl` fails with `E_DUPLICATE_DEFINITION` at its own `main`; with the Good one it builds. (The prelude already defines `square`, so a program-file `square` is itself `E_DUPLICATE_DEFINITION`.)

## Diagnosis

`E_DUPLICATE_DEFINITION` on your `main`, or `E_TOPLEVEL_STMTS_WITH_EXPLICIT_MAIN` in a file with no `main` of its own → check every module in the transitive `use` chain.

## See Also

- [test-program-library-tests-split](test-program-library-tests-split.md)
- [boot-module-build](boot-module-build.md)
