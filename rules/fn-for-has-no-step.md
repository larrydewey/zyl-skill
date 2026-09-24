# fn-for-has-no-step

> `for` has no step clause: the body must `set!` the loop variable, or the loop never ends.

## Why It Matters

`(for (bindings) cond body)` creates its variables, then runs `body` while `cond` holds. Nothing advances the variable for you. Forgetting the `set!` is an infinite loop. There is no iterator protocol: `for` does not walk collections.

## Bad

```lisp
(for (i 0) (< i 3)
  (print i))                      ; infinite loop
```

## Good

```lisp
(for ((i 0)) (< i 3)
  (begin
    (print i)
    (set! i (+ i 1))))

(for ((i 0) (j 100)) (< i 3)      ; several variables
  (begin
    (print (+ i j))
    (set! i (+ i 1))
    (set! j (- j 10))))
```

## Notes

- Accepted binding shapes: `((i 0))`, `((i 0) (j 10))`, `(i 0)` (short form), `()` (a plain while).
- Loop variables are `TMut`: `set!` on them is allowed.
- `while` and `for` are for effect: the spec says they return `Unit`; the implementation leaves the last body value. Do not use it.
- To iterate a `Vec`, index with `vec-get` up to `vec-len`; to iterate a `List`, recurse.

## See Also

- [fn-no-named-let-or-early-return](fn-no-named-let-or-early-return.md) - recursion idioms
- [data-collections-int-persistent](data-collections-int-persistent.md) - Vec indexing
