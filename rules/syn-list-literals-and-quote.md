# syn-list-literals-and-quote

> Build a `List` with `[a b c]` or `(list a b c)`, quote constant data with `'(1 2 3)`, and fill holes with `` `(... ,x ,@xs) ``; every element has one type, and a name inside quoted data is `E_MALFORMED_FORM`.

## Why It Matters

Since 2026-09-25 the reader has list sugar. All of these become the same `Cons` chain before type checking, built in source order, so elements are evaluated left to right and must share one type (a mix is `E_TYPE_MISMATCH` at the literal):

| Written | Reads as | Becomes |
|---|---|---|
| `[a b c]` | `(list a b c)` | `(Cons a (Cons b (Cons c Nil)))` |
| `'(1 (2 3))` | `(quote (1 (2 3)))` | a list literal of quoted elements: a `(List (List Int))` here |
| `` `(1 ,x ,@ys) `` | `(quasiquote ...)` | `(Cons 1 (Cons x (zyl-qq-append ys Nil)))` |

There is no symbol type, so quoted data holds only Int, Float, String and Bool atoms and lists of them. `'x`, `'(a b)` and a name outside an unquote in a quasiquote are `E_MALFORMED_FORM` ("a quoted list holds constant data; `a` is a name"). Code written from other Lisps that quotes symbols, or uses `` ` `` to build code, does not compile.

## Bad

```lisp
(print [1 "a"])                ; E_TYPE_MISMATCH: cannot unify Int with String
(print '(a b))                 ; E_MALFORMED_FORM: `a` is a name
(print '[1 2])                 ; E_MALFORMED_FORM: `list` is a name (quote of a bracket)
(print `(1 ,@y))               ; y is an Int: E_TYPE_MISMATCH, ,@ needs a List
(print `(,x `(,x)))            ; E_MALFORMED_FORM: nested quasiquote is not supported
(print `(1 . ,ys))             ; E_INVALID_CHAR at `.`: no dotted pairs
```

## Good

```lisp
(defn main ()
  (let x 3
    (let ys [4 5]
      (begin
        (print [1 2 3])                 ; [1, 2, 3]
        (print (list "a" "b"))          ; [a, b]
        (print '((1 2) (3)))            ; [[1, 2], [3]]
        (print `(0 ,x ,@ys 6))          ; [0, 3, 4, 5, 6]
        (print (list-length []))        ; 0
        (print (== [1 2] (Cons 1 (Cons 2 Nil))))  ; 1
        0))))
```

## Notes

- The result is the ordinary persistent `List` (`Cons`/`Nil`), so `match`, `list-length`, `list-reverse` and derived `Show` work on it. There are no vector or map literals: build a `Vec` or `Map` with its functions.
- `'5`, `'"s"` and `` `5 `` are just the atom.
- `[...]` inside a derive (`(derive T [Eq Show])`, `(:derive [Eq Show])`) still names traits; the reader's `list` head is dropped there.
- `,e` and `,@e` mean something only inside a quasiquote or a macro template; anywhere else they are `E_MALFORMED_FORM`. In a macro, `,@rest` splices a `&rest` parameter: see [macro-quasiquote-and-rest](macro-quasiquote-and-rest.md).
- `zyl-qq-append` is a `core/list` function of its own, so redefining `list-append` does not change what `,@` does.
- `print` of a list shows its elements with `Show`: a `List Bool` prints `[true, false]` although a bare Bool prints `1`.

## See Also

- [syn-no-stray-characters](syn-no-stray-characters.md) - which characters remain invalid
- [syn-brackets-and-balance](syn-brackets-and-balance.md) - how `[`, `(` and `{` are read
- [data-collections-persistent](data-collections-persistent.md) - List, Vec and Map
