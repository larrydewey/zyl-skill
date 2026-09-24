# type-annotations-guide-codegen

> Treat parameter annotations as documentation that also constrains inference; they are optional (inference usually finds the same type) and unchecked.

## Why It Matters

`(name Type)` unifies the parameter with `Type` during inference, and `Secret` feeds Secret tracking. Without it, inference usually infers the same type from use. Annotations are **not** checked: passing a Float to an `(a Int)` parameter compiles and computes on the bits (the conflict just makes those types unknown). There is no annotation for return types, `let` bindings or function types.

## Accepted annotation names

`Int`, `Float`, `Bool`, `String`, `Unit`, `Byte`, struct and ADT names, `Secret`, `(Secret Int)`. Unknown names (`Bogus`, an `alias` name) are silently accepted.

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

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [type-inference-does-not-reject](type-inference-does-not-reject.md)
