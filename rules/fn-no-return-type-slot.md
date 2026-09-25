# fn-no-return-type-slot

> Parameters are a bare name or `(name Type)`; there is no return-type annotation and no annotation on `let`.

## Why It Matters

Everything after the parameter list is the body. A return type written there is evaluated as an expression and fails with `E_UNBOUND_VARIABLE` ("unbound identifier `Int`"). Anything in the parameter list that is not a name or `(name Type)` is `E_MALFORMED_PARAMETER`, which is also how a dropped `)` that pulls the body into the parameter list gets caught (when no body form follows, the whole `defn` is `E_MALFORMED_FORM` instead). Annotations that used to mean nothing are errors since 2026-09-25: `(a : Int)` and a trait used as a type, `(a Ord)`, are `E_MALFORMED_PARAMETER`.

## Bad

```lisp
(defn add ((a Int) (b Int)) Int (+ a b))   ; `Int` read as body: E_UNBOUND_VARIABLE
(defn f (x (+ x 1)) x)                     ; E_MALFORMED_PARAMETER: `(+ ...)` is not a parameter
(defn id ((T) x) x)                        ; E_MALFORMED_PARAMETER
(defn g ((a : Int)) a)                     ; E_MALFORMED_PARAMETER: write (a Int)
(defn h ((a Ord)) a)                       ; E_MALFORMED_PARAMETER: `Ord` is a trait, not a type
```

## Good

```lisp
(defn add ((a Int) (b Int)) (+ a b))
(defn greet ((name String)) (print (str-concat "Hello, " name)))
(defn total ((xs (List Int))) (list-length xs))
(defn f (x) (+ x 1))
```

## Notes

- Annotation types: `Int`, `Float`, `Bool`, `String`, `Unit`, `Byte`, a struct/ADT name, an applied type such as `(List Int)` or `(Option String)`, `Secret` or `(Secret Int)`, and the runtime handle types.
- A capitalized name that is not a known type is a **type variable**, not an error: `(defn f ((v Bogus)) v)` is generic, and two parameters annotated `Bogus` must have the same type. A misspelled type name therefore makes the parameter polymorphic instead of failing. `alias` does not create a type name, so `(alias Num Int)` then `(x Num)` is also a type variable ([fn-unlowered-forms](fn-unlowered-forms.md)).
- Annotations are part of type inference: a call with a clashing argument is `E_TYPE_MISMATCH` with a note pointing at the declared parameter, and the annotation fixes the type code generation uses. See [type-annotations-constrain](type-annotations-constrain.md).
- Parameters are immutable (`TCap`); `set!` on one is `E_MUT_CONFLICT`.
- `defn` is the only function form: `defun` is not recognized, so a `defun` defines nothing and its callers get `E_UNBOUND_VARIABLE`.
- Duplicate parameter names are `E_DUPLICATE_PARAMETER`, except `_` and `_`-prefixed names.

## See Also

- [gen-no-type-parameter-syntax](gen-no-type-parameter-syntax.md) - no `(T)` groups
- [type-annotations-constrain](type-annotations-constrain.md) - what annotations do
- [syn-keywords-and-symbols](syn-keywords-and-symbols.md) - the `: Type` trap
