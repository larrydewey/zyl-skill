# macro-template-is-literal-code

> Write the macro body as the literal expansion, in exactly one form: parameters are replaced by argument source, and a backquote in a template builds list data, not code.

## Why It Matters

`(defmacro name (params) body)` (synonym `macro`) substitutes each parameter with the **unevaluated** argument expression and renames the body's own binders. Nothing in the body runs at compile time. The template is already code, so it needs no quasiquote: write `(if c 0 body)`, not `` `(if ,c 0 ,body) ``. A backquote in a template is an ordinary quasiquote *expression* that builds a list value at run time; a name in it outside an unquote (such as `if`) is `E_MALFORMED_FORM`. The body is exactly one form: `(defmacro m (e) a b)` is `E_MALFORMED_FORM` ("malformed `defmacro` form").

## Bad

```lisp
(defmacro my-unless (c body) `(if ,c 0 ,body))
;; error[E_MALFORMED_FORM]: malformed quasiquote: `if` is a name; write ,name for its value
(defmacro log2 (e)
  (print "evaluating")
  e)
;; error[E_MALFORMED_FORM]: malformed `defmacro` form
```

## Good

```lisp
(defmacro my-unless (c body) (if c unit body))
(defmacro my-when (c body) (my-unless (not c) body))   ; macros may use macros
(defmacro log-and-return (e)
  (begin (print "evaluating") e))
(defmacro framed (a &rest xs) `(,a ,@xs 0))            ; a quasiquote as list DATA: (framed 9 8) is [9, 8, 0]
```

## Notes

- `,x` in a template is accepted and means the same as `x`. `,@name` splices a `&rest` parameter; see [macro-quasiquote-and-rest](macro-quasiquote-and-rest.md).
- Parameters are plain identifiers, optionally ending in `&rest name`: no destructuring or literal patterns (a non-identifier parameter is `E_MALFORMED_PARAMETER`).
- A call must pass exactly one argument per fixed parameter (`E_ARITY_MISMATCH`), or at least that many when the list ends in `&rest`.
- The expansion is type-checked like hand-written code: `(if c 0 body)` with a `print` body is `E_TYPE_MISMATCH` (Int against Unit), so use `unit` for the empty branch of a statement macro.
- A macro cannot print its argument's source text; it can only place the argument where it will be evaluated.
- A macro cannot remove code at compile time: an `if` in the template is tested at run time.

## See Also

- [macro-quasiquote-and-rest](macro-quasiquote-and-rest.md)
- [macro-args-spliced](macro-args-spliced.md)
- [syn-no-stray-characters](syn-no-stray-characters.md)
