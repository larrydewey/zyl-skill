# err-partial-ops-spelled

> Call partial library functions by their spelled names: `name!` stops the program on bad input, `name?` returns an `Option`, and the plain name no longer exists (`vec-get!`/`vec-get?`, never `vec-get`). Spell your own panicking functions with a trailing `!`.

## Why It Matters

Spec and `docs/soundness.md` L8: the places a program can stop must be visible in its source. Every standard-library function that can panic ends in `!`, its total sibling ends in `?` and returns an `Option`, and there is no plain name. A call to an old name such as `vec-get` is `E_UNBOUND_VARIABLE`, with a did-you-mean listing the two spelled forms. `panic`, `assert`, the `assert-*` test forms and `unwrap` keep their names, so searching for `!`, `panic`, `assert` and `unwrap` finds every library call that can stop the program.

| Stops the program (`E_INDEX_OUT_OF_BOUNDS`) | Returns an `Option` |
|---|---|
| `vec-get!`, `vec-set!`, `vec-last!` | `vec-get?`, `vec-set?`, `vec-last?` |
| `slice-get!`, `slice-vec!`, `slice-sub!` | `slice-get?`, `slice-vec?`, `slice-sub?` |
| `view-slice!`, `view-sub!` | `view-slice?`, `view-sub?` |
| `i64x2-get!`/`-set!`, `i32x4-get!`/`-set!`, `u8x16-get!`/`-set!` | the same names with `?` |
| `option-expect!`, `result-expect!` (stop with your message) | `option-unwrap`, `result-unwrap` take a default instead |

`vec-get-or` and `slice-get-or` take a default and never stop. String slicing (`str-substring`, `view-take`, `view-drop`) is total by definition: it clamps to the overlap of the requested range with the string, so it has neither suffix.

The rule is enforced for definitions too, at the unused-binding stage (`unused_check.zyl`). A `defn` whose body calls `panic` (or the runtime's `zyl_panic`) directly and whose name has no trailing `!` is the warning `W_PANIC_UNMARKED` in a program and the error `E_PANIC_UNMARKED` in a program-facing standard-library module. `main`, `test` bodies and the compiler's own modules are exempt.

## Bad

```lisp
(use collections/vec)
(defn third (v) (vec-get v 2))   ; E_UNBOUND_VARIABLE: did you mean `vec-get!`, `vec-get?`
```

```lisp
(defn checked-age ((n Int)) (if (< n 0) (panic "negative age") n))   ; W_PANIC_UNMARKED
```

## Good

```lisp
(use collections/vec)
(defn third-or-zero (v)
  (match (vec-get? v 2)
    (Some x x)
    (None 0)))
(defn age! ((n Int)) (if (< n 0) (panic "negative age") n))
(defn main ()
  (let v (vec-push (vec-push (vec-new) 10) 20)
    (begin
      (print (third-or-zero v))    ; 0
      (print (vec-get! v 1))       ; 20
      (print (vec-get-or v 5 -1))  ; -1
      (print (age! 3))             ; 3
      (print (vec-get! v 2))       ; PANIC: E_INDEX_OUT_OF_BOUNDS: vec-get! index 2 is outside a Vec of 2 elements
      0)))
```

The last line prints a help line and a backtrace after the `PANIC:` line:

```text
PANIC: E_INDEX_OUT_OF_BOUNDS: vec-get! index 2 is outside a Vec of 2 elements
  = help: use `(vec-get? v i)` and match its Option, or check the index against `vec-len` first
  in vec-get!
  in main
```

## Notes

- Prefer `?` and a `match` wherever a bad index is an expected case; use `!` where it would be a bug, so the program stops at the cause.
- A `!` panic unwinds to the nearest `try` like any panic, and inside a `test` it fails that test.
- The check sees direct calls only: a plain-named function that panics through a callee is not flagged, but the callee is, which is where the `!` must then appear.
- Built-in forms and runtime primitives are outside the rule: exhausting a `fixed` region, running out of memory and an FFI timeout can stop a program without a `!` in sight. Integer division is spelled the same way (`div!`/`div?`, [fn-division-total](fn-division-total.md)).
- `vec-set?` returns `(Option Vec)`; an index equal to the length appends, as `vec-push` does, for both `vec-set!` and `vec-set?`.

## See Also

- [fn-division-total](fn-division-total.md)
- [err-no-assert-unwrap](err-no-assert-unwrap.md)
- [err-result-for-expected-failures](err-result-for-expected-failures.md)
- [data-views-and-slices](data-views-and-slices.md)
