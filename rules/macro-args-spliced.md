# macro-args-spliced

> Use each parameter once in a macro body, or bind it with `let` inside the body so the argument is evaluated exactly once.

## Why It Matters

Arguments are copied into the body as source. A parameter used twice evaluates its argument twice (side effects twice, cost twice); a parameter not reached is not evaluated at all. That laziness is the only reason to prefer a macro over a function.

## Bad

```lisp
(defmacro double (x) (+ x x))
(defn noisy () (begin (print "evaluated") 1))
(defn main () (begin (print (double (noisy))) 0))   ; prints "evaluated" twice, then 2
```

## Good

```lisp
(defmacro double (x) (let n x (+ n n)))       ; hygiene keeps `n` private
(defn square-fn (n) (* n n))
(defmacro square (x) (square-fn x))           ; or delegate to a function
(defn noisy () (begin (print "evaluated") 3))
(defn main () (begin (print (double (noisy))) (print (square (noisy))) 0))   ; each prints "evaluated" once: 6, 9
```

## Notes

- `&rest` arguments follow the same rule: every `,@body` in the template places every argument once more, so splice a rest parameter once.
- The expansion is type-checked like hand-written code, so a double evaluation is never hidden by a type error; it simply happens.

## See Also

- [macro-quasiquote-and-rest](macro-quasiquote-and-rest.md)

- [macro-hygiene](macro-hygiene.md)
- [macro-prefer-functions](macro-prefer-functions.md)
