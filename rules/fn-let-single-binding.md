# fn-let-single-binding

> `let` binds exactly one name; nest `let`s for several, and prefer the bare `(let name value body...)` spelling.

## Why It Matters

There is no multi-binding or parallel `let`, no `let*`, and no named `let`. The shapes other Lisps use now fail at compile time, but not always with a message that names the real problem:

- `(let ((x 1) (y 2)) ...)` is `E_MALFORMED_FORM` ("malformed `let` form").
- `(let (x 10 y 20) ...)` binds only `x`; `y` is `E_UNBOUND_VARIABLE` (and `x` gets `W_UNUSED_VARIABLE`).
- `(let loop ((n 10)) ...)` binds `loop` to the call `((n 10))`: `E_UNBOUND_VARIABLE` ("call to undefined function `n`").
- `let*` is an unknown function: `E_UNBOUND_VARIABLE`.

## Bad

```lisp
(let ((x 1) (y 2)) (+ x y))           ; E_MALFORMED_FORM
(let (x 10 y 20) (+ x y))             ; E_UNBOUND_VARIABLE: y
(let loop ((n 10)) ...)               ; named let: unsupported
```

## Good

```lisp
(let x 1
  (let y 2
    (+ x y)))                          ; 3; each let sees the ones outside it

(let x 10
  (print x)
  (print "done"))                      ; several body forms: an implicit begin
```

## Notes

- The bare form is what the stdlib and compiler use everywhere. The binding-list form `(let (x 10) body...)` also works and also takes several body forms (it used to keep only the first).
- There is no annotation on a `let`: `(let (x Int) 5 ...)` binds `x` to `Int`, an unbound identifier. Constrain the value instead (a typed helper, or an annotated parameter).
- A local `let` is monomorphic: a `let`-bound function value has one type for all its uses.
- `(let _ (side-effect) body)` runs an effect and discards its value.
- `let` copies one machine word: a second name for a `let-mut` value is an independent copy.
- Replace named-let loops with a top-level helper function (accumulator style) or `while`.

## See Also

- [own-let-mut-only-set](own-let-mut-only-set.md) - `let` vs `let-mut`
- [fn-no-named-let-or-early-return](fn-no-named-let-or-early-return.md) - loop and return idioms
- [fn-begin-multi-form-bodies](fn-begin-multi-form-bodies.md) - which bodies take several forms
