# macro-args-spliced

> Use each parameter once in a macro body, or bind it with `let` inside the body so the argument is evaluated exactly once.

## Why It Matters

Arguments are copied into the body as source. A parameter used twice evaluates its argument twice (side effects twice, cost twice); a parameter not reached is not evaluated at all. That laziness is the only reason to prefer a macro over a function.

## Bad

```lisp
(defmacro double (x) (+ x x))
(double (noisy))        ; prints "evaluated" twice
```

## Good

```lisp
(defmacro double (x) (let n x (+ n n)))       ; hygiene keeps `n` private
(defn square-fn (n) (* n n))
(defmacro square (x) (square-fn x))           ; or delegate to a function
```

## Notes

- `&rest` arguments follow the same rule: every `,@body` in the template places every argument once more, so splice a rest parameter once.
- The expansion is type-checked like hand-written code, so a double evaluation is never hidden by a type error; it simply happens.

## See Also

- [macro-quasiquote-and-rest](macro-quasiquote-and-rest.md)

- [macro-hygiene](macro-hygiene.md)
- [macro-prefer-functions](macro-prefer-functions.md)
