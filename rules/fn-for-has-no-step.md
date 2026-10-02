# fn-for-has-no-step

> `for` has no step clause: the body must `set!` the loop variable, or the loop never ends.

## Why It Matters

`(for (bindings) cond body)` creates its variables, then runs `body` while `cond` holds. Nothing advances the variable for you. Forgetting the `set!` is an infinite loop. There is no iterator protocol: `for` does not walk collections.

## Bad

```lisp
(defn main ()
  (begin
    (for ((i 0)) (< i 3)
      (print i))                  ; prints 0 forever: i never changes
    0))
```

## Good

```lisp
(defn main ()
  (begin
    (for ((i 0)) (< i 3)          ; 0 1 2
      (print i)
      (set! i (+ i 1)))
    (for ((i 0) (j 100)) (< i 3)  ; several variables: 100 91 82
      (print (+ i j))
      (set! i (+ i 1))
      (set! j (- j 10)))
    0))
```

## Notes

- Accepted binding shapes: `((i 0))`, `((i 0) (j 10))`, `(i 0)` (short form), `()` (a plain while).
- Loop variables are `TMut`: `set!` on them is allowed.
- `while` and `for` have type `Unit`: `(let r (for ...) (+ r 1))` is `E_TYPE_MISMATCH`. Collect results in a `let-mut` or use a recursive helper.
- The body may hold several forms (an implicit `begin`); the condition must be a `Bool`.
- To iterate a `Vec`, index with `vec-get!` up to `vec-len`; to iterate a `List`, recurse.

## See Also

- [fn-no-named-let-or-early-return](fn-no-named-let-or-early-return.md) - recursion idioms
- [data-collections-persistent](data-collections-persistent.md) - Vec indexing
