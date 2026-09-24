# macro-template-no-quasiquote

> Write the macro body as the literal expansion: parameters are replaced by argument source; there is no quasiquote, and only one body form is kept.

## Why It Matters

`(defmacro name (params) body)` (synonym `macro`) substitutes each parameter with the **unevaluated** argument expression and renames the body's own binders. Nothing in the body runs at compile time. Backquote, `,`, `,@` and `'` are not tokens: they **end the token stream** and silently drop the rest of the file (often surfacing as `undefined reference to _ZYL_main`). If the body has several forms, only the **last** is kept, silently.

## Bad

```lisp
(defmacro my-unless (c body) `(if ,c 0 ,body))   ; rest of file vanishes
(defmacro log2 (e)
  (print "evaluating")                          ; silently dropped
  e)
```

## Good

```lisp
(defmacro my-unless (c body) (if c 0 body))
(defmacro my-when (c body) (my-unless (not c) body))   ; macros may use macros
(defmacro log-and-return (e)
  (begin (print "evaluating") e))
```

## Notes

- Parameters are plain identifiers: no destructuring, rest or literal patterns (non-identifier parameter is `E_MALFORMED_PARAMETER`).
- A call must pass exactly one argument per parameter (`E_ARITY_MISMATCH`).
- A macro cannot print its argument's source text; it can only place the argument where it will be evaluated.
- A macro cannot remove code at compile time: `(if (debug-enabled) body 0)` is tested at run time.

## See Also

- [macro-args-spliced](macro-args-spliced.md)
- [syn-no-stray-characters](syn-no-stray-characters.md)
