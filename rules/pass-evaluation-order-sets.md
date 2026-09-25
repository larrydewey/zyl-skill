# pass-evaluation-order-sets

> When a lowering or codegen shortcut reads a local out of order (late, or straight into a register), guard it with `icnf-has-set` / `icnf-sets`: only a `set!` inside the expression itself can change a local while it is evaluated.

## Why It Matters

Evaluation is strictly left to right. `(+ x (begin (set! x 10) x))` with `x` = 1 is 11: the left operand is read **before** the right one runs. A backend that keeps `x` in a register and reads the register when it computes the sum would get 20. Since 4892ede every fast path that delays or reorders a local read checks first whether a later sibling could change it.

The check is cheap and exact because of two language rules: a closure cannot `set!` a captured `let-mut` (`E_MUT_CONFLICT`), and `set!` on anything but a `let-mut` local is rejected. So a call, a closure call or an FFI call cannot change a caller's local; only an `ISet` in the expression tree can. `icnf.zyl` provides:

| Helper | Answers |
|---|---|
| `(icnf-has-set e)` | does `e` contain any `ISet` (total over `Icnf`) |
| `(icnf-list-has-set es)` | does any of `es` |
| `(icnf-sets e n)` | does `e` contain a `set!` of `n` |

Users: the native backend (`ml-operand` copies a local read into a fresh vreg when a later operand or argument has a set; `ml-fresh-args` skips the move of an unchanged parameter only when no later argument sets anything; `ml-views` loads a byte buffer's data pointer once only for a parameter no `set!` targets), the stack machine's direct-register call path (`cg-direct-args-ok` requires no set in any argument), and copy propagation (`opt-sets-or-binds`).

## Bad

```lisp
; a shortcut that evaluates the complex argument first and reads locals afterwards,
; without the guard: (f x (begin (set! x 2) x)) would pass 2 twice
(if (<= (cg-count-complex env args) 1) (cg-direct-args st env args n) ...)
```

## Good

```lisp
(defn cg-direct-args-ok (env args n)
  (and (<= n 6) (and (<= (cg-count-complex env args) 1) (not (icnf-list-has-set args)))))
```

## Notes

- A new `Icnf` node needs a case in `icnf-has-set` (a total match) and in `icnf-sets`.
- If the language ever lets a closure or a callee change a caller's local, every one of these guards becomes wrong at once.
- Verified 2026-09-25 in both backends: `(+ x (begin (set! x 10) x))` from 1 is 11, and `(add3 x (begin (set! x (* x 2)) x) x)` from 3 is 15.

## See Also

- [det-left-to-right](det-left-to-right.md)
- [cg-call-arg-staging](cg-call-arg-staging.md)
- [cg-native-backend-mir](cg-native-backend-mir.md)
