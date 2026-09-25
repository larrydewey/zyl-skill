# macro-prefer-functions

> Use a macro only when you need call-by-name evaluation (skip or repeat an argument) or new binding syntax; otherwise write a function.

## Why It Matters

Functions evaluate arguments once, can be passed to higher-order functions, show up in stack traces and hover, and get one checked type; a macro's expansion is type-checked separately at every call site, so a type error lands inside the expanded code. Macros duplicate or drop argument evaluation, cannot be passed as values, and have no expansion viewer (no `--emit-expanded`, no `:macroexpand`).

## Good uses

```lisp
(defmacro my-unless (c body) (if c unit body))        ; skips body
(defmacro swap! (a b) (let tmp a (begin (set! a b) (set! b tmp))))  ; binds names
(defmacro my-when (c &rest body) (if c (begin ,@body) unit))  ; new syntax: a block of statements
```

## Built-ins that look like macros

`and`, `or`, `cond`, `not`, `begin` are core forms recognized by `convert-ast` before macro expansion. There is no `let*` and no `when`/`unless` form (`when` is only a keyword inside literal-match guards; the prelude `when`/`unless` in `core/core.zyl` are eager functions whose body must be `Unit`).

## Debugging

Write the expected expansion by hand and compare behavior; test the macro inside a `(test ...)` form; read `--emit-asm`.

## See Also

- [macro-args-spliced](macro-args-spliced.md)
- [macro-shadows-functions](macro-shadows-functions.md)
