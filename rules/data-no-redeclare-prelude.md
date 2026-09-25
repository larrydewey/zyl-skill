# data-no-redeclare-prelude

> Never redeclare `Option`, `Result`, `List`, their constructors, or any prelude function name; pick another name.

## Why It Matters

`core/core` (which loads `core/show`, `core/option`, `core/result` and `core/list`) is injected into every program that does not already load one of them. Its names are taken: a second `(deftype Option ...)` or a `(defn compose ...)` is `E_DUPLICATE_DEFINITION`, and a constructor of your own named `Some`, `None`, `Ok`, `Err`, `Cons` or `Nil` is `E_DUPLICATE_VARIANT: constructor `Some` is already a prelude constructor`, because the standard library uses those names unqualified.

## Bad

```lisp
(deftype Option (Some T) None)   ; E_DUPLICATE_DEFINITION: type `Option`
(deftype Mb (Some T) (Nope))     ; E_DUPLICATE_VARIANT: prelude constructor
(defn compose (f g) ...)         ; E_DUPLICATE_DEFINITION
(defn min (a b) ...)             ; E_DUPLICATE_DEFINITION
```

## Good

```lisp
(deftype Maybe (Just T) (Nothing))
(defn compose2 (f g) (fn (x) (f (g x))))
(defn smaller (a b) (if (< a b) a b))
```

## Taken names (prelude)

- Types/constructors: `Option`/`Some`/`None`, `Result`/`Ok`/`Err`, `List`/`Cons`/`Nil`.
- `core/core`: `identity const flip compose apply abs max min clamp signum square cube xor nand nor implies when unless is-bool is-zero is-even is-odd print-int print-float print-string print-bool option-to-result option-from-result result-to-option result-from-option`.
- `core/option`: `option-some option-none option-is-some option-is-none option-unwrap option-unwrap-or option-expect option-map option-flatmap option-and option-or option-inspect`.
- `core/result`: `result-ok result-err result-is-ok result-is-err result-unwrap result-unwrap-or result-expect result-map result-flatmap result-and-then result-or-else result-and result-or result-inspect`.
- `core/list`: `is-nil list-car list-cdr list-rest car cdr cadr caddr cddr list-length list-append list-reverse list-reverse-acc list-sum`, plus the helpers `zyl-qq-append` (quasiquote splicing), `list-show-items`, `list-debug-items`, `list-compare`, `list-hash`.
- `core/show`: traits `Show Debug Eq Ord Hash Clone Secret` and helpers `show-ord-int show-mix show-str-hash`.

`use`ing other stdlib modules adds more (the stdlib is one fully visible surface), e.g. `list-map` and `list-nth` from `collections/collections`.

## Notes

- A `defmacro` may deliberately shadow a prelude function (e.g. a lazy `unless`); the macro then takes over every call.
- A `(trait Show ...)` of your own is accepted silently; do not rely on it replacing the prelude's.

## See Also

- [reference: stdlib](../references/stdlib.md)
- [macro-shadows-functions](macro-shadows-functions.md)
