# type-annotations-guide-codegen

> Use parameter annotations to tell code generation what a value is, not as a safety net; they are optional and unchecked.

## Why It Matters

`(name Type)` annotations change how a parameter is printed, compared (`=` on strings), and used in arithmetic (SSE for Float), and they feed Secret tracking. They are **not** checked against call sites or even against known type names. There is no annotation for return types, `let` bindings or function types.

## Accepted annotation names

`Int`, `Float`, `Bool`, `String`, `Unit`, `Byte`, struct and ADT names, `Secret`, `(Secret Int)`. Unknown names (`Bogus`, an `alias` name) are silently accepted.

## Good

```lisp
(defn half ((x Float)) (/ x 2.0))
(defn greet ((name String)) (print (str-concat "Hello, " name)))
(defn apply-twice (f x) (f (f x)))       ; function-typed params stay unannotated
```

## Notes

- `Vec<T>` / `Map<K,V>` are notation only: `<` and `>` are identifier characters, so `Vec<Int>` lexes as one identifier.
- Function types (`TFun`) exist only inside the inferer.
- `(alias Name Type)` has no effect.

## See Also

- [fn-annotate-string-float-params](fn-annotate-string-float-params.md)
- [type-inference-does-not-reject](type-inference-does-not-reject.md)
