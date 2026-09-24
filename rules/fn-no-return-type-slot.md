# fn-no-return-type-slot

> Parameters are a bare name or `(name Type)`; there is no return-type annotation and no annotation on `let`.

## Why It Matters

Everything after the parameter list is the body. A return type written there is evaluated as an expression and fails with `E_UNBOUND_VARIABLE`. Anything in the parameter list that is not a name or `(name Type)` is `E_MALFORMED_PARAMETER`, which is also how a dropped `)` that pulls the body into the parameter list gets caught.

## Bad

```lisp
(defn add ((a Int) (b Int)) Int (+ a b))   ; `Int` read as body: E_UNBOUND_VARIABLE
(defn f (x (+ x 1)))                       ; E_MALFORMED_PARAMETER
(defn id ((T) x) x)                        ; E_MALFORMED_PARAMETER
```

## Good

```lisp
(defn add ((a Int) (b Int)) (+ a b))
(defn greet ((name String)) (print (str-concat "Hello, " name)))
(defn f (x) (+ x 1))
```

## Notes

- Annotation type names: `Int`, `Float`, `Bool`, `String`, `Unit`, `Byte`, a struct/ADT name, `Secret` or `(Secret Int)`. Unknown names such as `(v Bogus)` are accepted silently.
- Annotations are checked only at direct calls, for definite clashes (`E_TYPE_MISMATCH`, see [type-inference-does-not-reject](type-inference-does-not-reject.md)), and they constrain the inferred types code generation follows (see [fn-types-drive-codegen](fn-types-drive-codegen.md)).
- Parameters are immutable (`TCap`); `set!` on one is `E_MUT_CONFLICT`.
- `defn` is the canonical form; `defun` is not reliably recognized (the book's chapters disagree) — do not use it.
- Duplicate parameter names are `E_DUPLICATE_PARAMETER`, except `_` and `_`-prefixed names.

## See Also

- [gen-no-type-parameter-syntax](gen-no-type-parameter-syntax.md) - no `(T)` groups
- [type-annotations-guide-codegen](type-annotations-guide-codegen.md) - what annotations do
