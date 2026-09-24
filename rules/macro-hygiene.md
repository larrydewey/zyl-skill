# macro-hygiene

> Pass everything a macro needs from the call site as an argument; the body's binders are renamed and it cannot see the caller's locals.

## Why It Matters

Hygiene is automatic and two-sided:

1. **Body binders are renamed** per expansion to `name__hygN` (`let`, `let-mut`, `fn`/`defn` params, `for`, `catch` name, `with-resource`, match pattern variables). `N` is a source-order counter, so expansion is deterministic. `_` is never renamed. Arguments keep their names and still refer to the caller's variables.
2. **Free names resolve where the macro is defined** (module resolution has already turned top-level references into canonical keys). A body naming a variable that is unbound at the definition but local at the call site is **rejected** with `E_UNBOUND_VARIABLE`, not captured.

## Bad

```lisp
(defmacro getv () v)
(let v 3 (print (getv)))
;; error[E_UNBOUND_VARIABLE]: macro `getv` refers to `v`, which is not bound where the macro is defined
```

## Good

```lisp
(defmacro add-tmp (e) (let tmp 100 (+ tmp e)))
(let tmp 1 (print (add-tmp tmp)))       ; 101: expands to (let tmp__hyg0 100 (+ tmp__hyg0 tmp))

(defmacro swap! (a b) (let tmp a (begin (set! a b) (set! b tmp))))
(let-mut tmp 1 (let-mut y 2 (begin (swap! tmp y) (print tmp) (print y))))   ; 2 1
```

## Notes

- A diagnostic naming `something__hygN` refers to a macro-bound variable.
- Where a parameter supplies a binder or `set!` target, the caller's identifier is used, so a macro can bind or assign a variable the caller names.

## See Also

- [macro-name-positions](macro-name-positions.md)
- [det-left-to-right](det-left-to-right.md) - deterministic expansion
