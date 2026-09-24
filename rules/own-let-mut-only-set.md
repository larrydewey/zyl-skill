# own-let-mut-only-set

> `set!` only a plain name bound by `let-mut` (or a `for` variable) in the current scope; everything else is immutable.

## Why It Matters

The TMut/TCap aliasing invariant (spec §10) is enforced by name in `mutability_check.zyl`: `let` bindings and parameters are `TCap`, `let-mut` and `for` variables are `TMut`, and `set!` is the only mutation. A `set!` on anything else is `E_MUT_CONFLICT`. `set!` **rebinds** the name to a new word; it never mutates the old value in place.

## Bad

```lisp
(defn inc (n) (set! n (+ n 1)))       ; parameter: E_MUT_CONFLICT
(let x 1 (set! x 2))                  ; let: E_MUT_CONFLICT
(set! (struct-get p "x") 5)           ; field: E_MUT_CONFLICT (parser)
```

## Good

```lisp
(defn inc (n) (+ n 1))                ; return the new value

(defn total (v)
  (let-mut sum 0
    (begin
      (for (i 0) (< i (vec-len v))
        (begin
          (set! sum (+ sum (vec-get v i)))
          (set! i (+ i 1))))
      sum)))
```

## Capability table

| You write | Capability | `set!`? |
|---|---|---|
| `(let x v body)` | TCap | no |
| `(let-mut x v body)` | TMut | yes |
| function parameter | TCap | no |
| `for` loop variable | TMut | yes |

## Notes

- Use `let-mut` sparingly; prefer immutable `let` and recursion.
- These diagnostics print as unlocated `PANIC: E_MUT_CONFLICT: ...` lines (except the closure-capture case, which is located).
- Capabilities are never written in source; there is no syntax to annotate a parameter `TMut`.

## See Also

- [own-no-closure-captured-mutation](own-no-closure-captured-mutation.md)
- [data-struct-immutable-rebind](data-struct-immutable-rebind.md)
