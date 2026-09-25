# closure-no-recursive-lambda

> Write recursive helpers as top-level `defn`s; a lambda cannot refer to itself.

## Why It Matters

There is no named `let`, no `letrec`, no mutable cell to tie a knot. A `fn` referring to the name it is being bound to sees an unbound (or outer) name: `(let f (fn (n) (if (= n 0) 0 (f (- n 1)))) ...)` is `E_UNBOUND_VARIABLE` ("call to undefined function `f`").

## Good

```lisp
(defn fact (n) (if (<= n 1) 1 (* n (fact (- n 1)))))
(defn apply-to-list (xs) (list-map (fn (x) (fact x)) xs))  ; lambda calls the defn; (use collections/collections)
```

## Notes

- Prelude combinators already exist: `identity`, `const`, `flip` (`(flip f a b)` calls `(f b a)`), `compose`, `apply`. Defining your own with those names is `E_DUPLICATE_DEFINITION`; `partial` is not in the prelude (write `(defn partial2 (f x) (fn (y) (f x y)))`).
- `collections/collections` has `list-map f xs`, `list-filter pred xs`, `list-fold f acc xs` (`(list-fold (fn (acc x) (+ acc x)) 0 [1 2 3])` is 6).
- Operators are not function values: `(flip - 1 10)` is `E_UNBOUND_VARIABLE`; write `(flip (fn (a b) (- a b)) 1 10)`.

## See Also

- [closure-explicit-fn-syntax](closure-explicit-fn-syntax.md)
- [data-no-redeclare-prelude](data-no-redeclare-prelude.md)
