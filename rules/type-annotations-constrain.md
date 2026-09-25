# type-annotations-constrain

> Annotate a parameter as `(name Type)` to fix its type: it unifies with the parameter everywhere (top-level functions and lambdas alike), so any clash is `E_TYPE_MISMATCH`; annotations are optional where use already determines the type.

## Why It Matters

`(name Type)` is a real constraint in the sound checker ([type-sound-checking](type-sound-checking.md)). A clashing argument is reported at the argument and labelled at the parameter (`parameter `a` of `add` is declared `Int``). `Secret` also feeds Secret tracking. Annotations are how you pin a type that use leaves open, document an interface, and turn a confusing error deep inside a body into one at the call. There is no annotation for return types or `let` bindings.

## Accepted annotation names

| Written | Meaning |
|---|---|
| `Int`, `Float`, `Bool`, `String`, `Unit` | the primitives |
| `ByteBuf`, `ByteSlice` | byte-buffer handles |
| `Arena`, `Ptr`, `Words`, `StrBuf`, `UF`, `Actor`, `Fd`, `FileId`, `FnPtr` | opaque runtime handles (`compiler/ffi_sigs.zyl`); `arena-create` returns an `Arena` |
| `(Array a)`, `(Ref a)`, `(Pin a)`, `(SMap v)`, `(WVec v)`, `(Attr k v)` | parameterized runtime handles |
| a struct or ADT name, bare or applied: `Vec`, `(Vec String)`, `(List Int)`, `(Map String Val)`, `(Box String)` | that type |
| `(Fn (A B) R)` | a function from `A B` to `R` |
| `Secret`, `(Secret Int)` | a secret value |

- An unknown **uppercase** name (`T`, `Bogus`, `Byte`, an `alias` name) is a type variable scoped to that one signature: `((a T) (b T))` forces both arguments to one type, and the function stays polymorphic. `Byte` is not a type.
- An unknown lowercase name is a fresh, unconstrained variable per occurrence.
- A trait name (`(a Ord)`) is `E_MALFORMED_PARAMETER: `Ord` is a trait, not a type`; `(a : Int)` is `E_MALFORMED_PARAMETER` too (write `(a Int)`).

## Good

```lisp
(defn half ((x Float)) (/ x 2.0))
(defn greet ((name String)) (print (str-concat "Hello, " name)))
(defn apply-twice ((f (Fn (Int) Int)) (x Int)) (f (f x)))
(defn count-strings ((v (List String))) (list-length v))

(apply-twice (fn (y) (str-length y)) 3)
;; E_TYPE_MISMATCH: expected `(Int -> Int)`, found `(String -> Int)`
```

## Notes

- `Vec<T>` is notation only: `<` and `>` are identifier characters. Write `(Vec T)`.
- `(alias Name Type)` has no effect; `Name` is then just an unknown uppercase name (a type variable).
- `Bool` params reject `1`/`0`: pass `true`/`false`.
- Hover in the language server shows a parameter's annotated type by its source name.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [type-sound-checking](type-sound-checking.md)
- [data-field-types](data-field-types.md)
