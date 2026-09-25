# macro-shadows-functions

> Remember that a macro takes over every call to a same-named imported function, program-wide.

## Why It Matters

Module resolution gives a macro the canonical key of a same-named imported function; the macro then expands at every call site, including calls that would have reached the function. This is the standard way to get a lazy `when`/`unless` over the prelude's eager functions — and a surprise if done by accident.

## Good

```lisp
(defmacro unless (c body) (if (not c) body unit))
(defn main ()
  (begin
    (unless true (print "should not print"))   ; not printed
    (print "end")
    0))
```

## Notes

- Without the macro, prelude `unless` is a function: both arguments are evaluated and the body prints.
- A macro and a `defn` with the same name in the same file is `E_DUPLICATE_DEFINITION` ("function `unless` has the same name as a macro").
- The else branch must have the body's type: `(if (not c) body 0)` around a `print` is `E_TYPE_MISMATCH` (Unit against Int).

## See Also

- [macro-prefer-functions](macro-prefer-functions.md)
- [fn-conditionals](fn-conditionals.md)
