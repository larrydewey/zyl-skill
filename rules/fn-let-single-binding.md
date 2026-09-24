# fn-let-single-binding

> `let` binds exactly one name; nest `let`s for several, and prefer the bare `(let name value body)` spelling.

## Why It Matters

There is no multi-binding or parallel `let`, no `let*`, and no named `let`. The shapes other Lisps use are accepted in ways that silently produce the wrong program:

- `(let (x 10 y 20) ...)` binds only `x`; `y` is then `E_UNBOUND_VARIABLE`.
- `(let ((x 1) (y 2)) ...)` is **not rejected** and compiles to the wrong program.
- The binding-list form `(let (x 10) body...)` takes **exactly one** body form; anything after the first is silently dropped.

## Bad

```lisp
(let ((x 1) (y 2)) (+ x y))           ; wrong program, no diagnostic
(let (x 10) (print x) (print "done")) ; second print silently dropped
(let loop ((n 10)) ...)               ; named let: unsupported
```

## Good

```lisp
(let x 1
  (let y 2
    (+ x y)))                          ; 3; each let sees the ones outside it

(let x 10
  (begin (print x) (print "done")))
```

## Notes

- The bare form is what the stdlib and compiler use everywhere; it takes any number of body forms (but see [fn-begin-multi-form-bodies](fn-begin-multi-form-bodies.md)).
- `(let _ (side-effect) body)` runs an effect and discards its value.
- `let` copies one machine word: a second name for a `let-mut` value is an independent copy.
- Replace named-let loops with a top-level helper function (accumulator style) or `while`.

## See Also

- [own-let-mut-only-set](own-let-mut-only-set.md) - `let` vs `let-mut`
- [fn-no-named-let-or-early-return](fn-no-named-let-or-early-return.md) - loop and return idioms
