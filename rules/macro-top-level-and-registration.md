# macro-top-level-and-registration

> Define macros only at top level, once per name; they may be used before their definition.

## Why It Matters

All top-level `defmacro`s are collected before any expansion (`me-collect`), then stripped from the program. A `defmacro` inside a function or any other form is `E_MACRO_ILLEGAL_ACCESS` (its template could name run-time variables). Two macros with one name, or a macro and a function with one name **in the same file**, is `E_DUPLICATE_DEFINITION`.

## Good

```lisp
(defn main () (begin (print (triple 7)) 0))   ; used before definition: 21
(defmacro triple (x) (+ x (+ x x)))
```

## Notes

- Macro calls expand everywhere: function and test bodies, `let`, `if`, `while`, `for`, match arms, `fn` bodies, `try`, impl methods, and at top level (where a macro can expand to a definition).
- Expansion is innermost-first: arguments expand before the macro receiving them, then the result is expanded again.
- The REPL accepts `defmacro` for interactive experiments.

## See Also

- [macro-shadows-functions](macro-shadows-functions.md)
