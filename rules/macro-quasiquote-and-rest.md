# macro-quasiquote-and-rest

> Take a variable number of macro arguments with `&rest name` and splice them with `,@name` only where a form takes any number of expressions; to build a list from them, use `` `(,@name) ``, never `(list ,@name)` or `[... ,@name]`.

## Why It Matters

A parameter list may end in `&rest name`. The remaining arguments bind to `name`, and the template uses them two ways:

1. **`,@name`** splices them in place, in order. The splice changes how many children a node has, so it is allowed only where any number may appear: a call's arguments, `begin`, `print`, `ffi-call` arguments. Anywhere else (an `if`, which has exactly three parts; a bare `,@xs` as the whole template) it is `E_MALFORMED_FORM` ("`,@` splices only where any number of expressions may appear").
2. **`name` alone** is the list of the arguments, so they must all have one type: `(list-length xs)` counts them, and a quasiquote in the template can splice them into list data.

`(list ...)` and `[...]` are read as chains of two-field `Cons` constructors before macros expand, so `(list ,@xs)` and `[0 ,@xs]` try to splice into a constructor's fields and are `E_MALFORMED_FORM` ("`,@` cannot splice into a constructor's fields"). A quasiquote builds the list instead.

Spliced arguments are the caller's code and keep their names: hygiene still renames only the template's own binders.

## Bad

```lisp
(defmacro bad (c &rest body) (if c ,@body 0))      ; E_MALFORMED_FORM: an if has fixed parts
(defmacro as-list (&rest xs) (list ,@xs))          ; E_MALFORMED_FORM: splice into Cons fields
(defmacro whole (&rest xs) ,@xs)                   ; E_MALFORMED_FORM: not inside a list of expressions
(defmacro m (a &rest) a)                           ; E_MALFORMED_PARAMETER: &rest needs one name, last
(defmacro one-plus (a &rest b) (+ a 1))
(one-plus)                                         ; E_ARITY_MISMATCH: takes at least 1 argument(s)
```

## Good

```lisp
(defmacro my-when (c &rest body) (if c (begin ,@body) unit))
(defmacro sum (&rest xs) (+ 0 ,@xs))                ; (sum) is (+ 0), 0
(defmacro count-args (&rest xs) (list-length xs))   ; (count-args 5 6 7) is 3
(defmacro as-list (&rest xs) `(,@xs))               ; (as-list 1 2) is [1, 2]
(defmacro framed (a &rest xs) `(,a ,@xs 0))         ; (framed 9 8 7) is [9, 8, 7, 0]

(defn main ()
  (begin
    (my-when true (print "one") (print "two"))
    (print (sum 1 2 3))                             ; 6
    0))
```

## Quasiquote outside macros

`` `d `` is `(quasiquote d)`: quoted data with holes, usable in any expression. `,e` puts the value of `e` in the list, `,@e` every element of the list `e`: `` `(1 ,x ,@ys) `` is `(Cons 1 (Cons x (zyl-qq-append ys Nil)))`, parts evaluated left to right, all of one element type (`E_TYPE_MISMATCH` otherwise). A name outside an unquote (`` `(1 x) ``), a quasiquote nested inside another, and a `,` or `,@` outside both a quasiquote and a macro template are `E_MALFORMED_FORM`.

## See Also

- [macro-template-is-literal-code](macro-template-is-literal-code.md)
- [macro-args-spliced](macro-args-spliced.md)
- [macro-hygiene](macro-hygiene.md)
