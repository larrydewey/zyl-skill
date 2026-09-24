# fn-begin-multi-form-bodies

> Wrap every body that has more than one form in `begin`.

## Why It Matters

`defn`, `let` and `test` accept several forms directly, but then a `let` that is one of those forms **stays in scope for the forms after it**. Shadowing then leaks: the outer name is replaced for the rest of the body. `begin` gives the scoping the language defines. `if` branches and `match` arm bodies take exactly one expression, so a multi-step branch *must* be a `begin`.

## Bad

```lisp
(defn main ()
  (let x 10
    (let x 20 (print x))   ; prints 20
    (print x)))            ; ALSO prints 20: the inner let leaked
```

## Good

```lisp
(defn main ()
  (let x 10
    (begin
      (let x 20 (print x)) ; 20
      (print x)            ; 10
      0)))
```

## Notes

- `(begin e1 ... en)` evaluates left to right; its value is `en`. An empty `begin` is 0.
- A `catch` handler is one expression: use `begin` or call a function.
- Macro bodies keep only their last form, so macros also need `begin` for several steps.

## See Also

- [fn-let-single-binding](fn-let-single-binding.md) - the other `let` trap
- [macro-template-no-quasiquote](macro-template-no-quasiquote.md) - one body form per macro
