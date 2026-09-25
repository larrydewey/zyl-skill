# macro-no-recursion

> Never write a macro that can expand into a call of itself, even behind an `if`; put recursion in functions.

## Why It Matters

The body is never evaluated at expansion time, so a guarding `if` does not stop expansion. The expander reports `E_MACRO_NON_TERMINATION` as soon as a macro is called while its own expansion is in progress, directly or through other macros, and also when distinct macros nest more than 256 deep.

## Bad

```lisp
(defmacro countdown (n) (if (= n 0) 0 (countdown (- n 1))))
(defmacro ping (x) (pong x))
(defmacro pong (x) (ping x))
```

## Good

```lisp
(defn countdown (n) (if (= n 0) 0 (countdown (- n 1))))
```

## See Also

- [macro-template-is-literal-code](macro-template-is-literal-code.md)
- [macro-quasiquote-and-rest](macro-quasiquote-and-rest.md)
