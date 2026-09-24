# closure-no-recursive-lambda

> Write recursive helpers as top-level `defn`s; a lambda cannot refer to itself.

## Why It Matters

There is no named `let`, no `letrec`, no `TBox` to tie a knot. A `fn` referring to the name it is being bound to sees an unbound (or outer) name.

## Good

```lisp
(defn fact (n) (if (<= n 1) 1 (* n (fact (- n 1)))))
(defn apply-to-list (xs) (my-map (fn (x) (fact x)) xs))   ; lambda calls the defn
```

## Notes

- Prelude combinators already exist: `identity`, `const`, `flip` (`(flip f a b)` calls `(f b a)`), `compose`, `apply`. Defining your own with those names is `E_DUPLICATE_DEFINITION`; `partial` is not in the prelude (write `(defn partial2 (f x) (fn (y) (f x y)))`).
- `collections/collections` has `list-map f xs`, `list-filter pred xs`, `list-fold f acc xs`.

## See Also

- [closure-explicit-fn-syntax](closure-explicit-fn-syntax.md)
- [data-no-redeclare-prelude](data-no-redeclare-prelude.md)
