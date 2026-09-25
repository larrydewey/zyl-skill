# closure-explicit-fn-syntax

> Write anonymous functions as `(fn (params) body)` or `(lambda (params) body)`; there is no shorthand.

## Why It Matters

`((x) (* x x))` is not a lambda: it is a call whose head `(x)` is itself a call to `x`, reported as `E_UNBOUND_VARIABLE` with a hint pointing at `fn`. Closures are ordinary values: bind with `let`, pass, store in data, return, call through a computed head.

## Good

```lisp
(defn make-adder (n) (fn (x) (+ x n)))
(defn apply-twice (f x) (f (f x)))

(defn main ()
  (let square (fn (x) (* x x))
    (let add5 (make-adder 5)
      (begin
        (print (square 5))                        ; 25
        (print (apply-twice square 3))            ; 81
        (print (apply-twice (fn (x) (+ x 1)) 5))  ; 7
        (print ((make-adder 10) 5))               ; 15: computed head is fine
        (print (add5 10))                         ; 15
        0))))
```

## Notes

- `fn` and `lambda` are the same form; neither takes a name.
- Any number of parameters; bodies may contain `match`, `try`, nested lambdas.
- A function-typed parameter may be left unannotated (inference finds its type) or annotated `(f (Fn (Int) Int))`: `(Fn (A ...) R)` is the function type, also used for extern callbacks.
- Operators are not values: `(apply-twice + 1)` is `E_UNBOUND_VARIABLE` for `+`; wrap them, `(fn (a b) (+ a b))`.
- Bodies are implicit `begin`s: `(fn (x) (print x) x)` runs both forms (older compilers kept only the last).
- A top-level closure value can be a `(def name (fn ...))`; `defn` is the usual form.

## See Also

- [closure-capture-by-value](closure-capture-by-value.md)
- [closure-no-recursive-lambda](closure-no-recursive-lambda.md)
