# type-annotations-guide-codegen

> Treat parameter annotations as documentation that constrains inference and is checked at direct calls: a definitely clashing argument is `E_TYPE_MISMATCH`; they are optional (inference usually finds the same type).

## Why It Matters

`(name Type)` unifies the parameter with `Type` during inference, and `Secret` feeds Secret tracking. Without it, inference usually infers the same type from use. At a call to a top-level function, an argument whose inferred type definitely clashes with the annotation (`(add 1.5 2.0)` for `(a Int)`, `(Cons "a" Nil)` for `(xs (List Int))`) is `E_TYPE_MISMATCH`, labelled at the parameter. Unknown/type-variable parts and `Unit` never clash; lambda params and trait-method calls are not checked. There is no annotation for return types, `let` bindings or function types.

## Accepted annotation names

`Int`, `Float`, `Bool`, `String`, `Unit`, `Byte`, struct and ADT names, `Secret`, `(Secret Int)`. Unknown names (`Bogus`, an `alias` name) are silently accepted as type variables (never clash).

## Good

```lisp
(defn half ((x Float)) (/ x 2.0))
(defn greet ((name String)) (print (str-concat "Hello, " name)))
(defn apply-twice (f x) (f (f x)))       ; function-typed params stay unannotated
```

## Notes

- Applied types are written as lists: `(v (Vec String))`, `(m (Map String Val))`. `Vec<T>` is notation only: `<` and `>` are identifier characters.
- Function types (`TFun`) exist only inside the inferer.
- `(alias Name Type)` has no effect.
- `Bool` params reject `1`/`0`: pass `true`/`false`.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [type-inference-does-not-reject](type-inference-does-not-reject.md)
