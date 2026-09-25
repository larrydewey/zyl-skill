# trait-derive-show

> Derive Show, Debug, Eq, Ord, Hash and Clone with `(derive T ...)`; every field type must implement the trait.

## Why It Matters

`compiler/derive.zyl` expands `(derive T Show Eq ...)` (or `(derive T [Show Eq])`) into one `(impl Trait T ...)` per trait, calling the trait's method on each field (instantiated per field type). The six derivable traits (spec 5.6), all declared in the prelude (`core/show`):

| Trait | Method | Derived behavior |
|---|---|---|
| `Show` | `(Show.show x)` | variant `Name(f, ...)`, struct `Name { f: v, ... }`; strings unquoted; `print` of the value uses it |
| `Debug` | `(Debug.debug x)` | same shape, strings quoted: `Q { n: 1, s: "a" }` |
| `Eq` | `(Eq.eq a b)` | structural equality, field by field |
| `Ord` | `(Ord.compare a b)` | `-1`/`0`/`1`: variant declaration order first, then fields lexicographically |
| `Hash` | `(Hash.hash x)` | deterministic `Int` (FNV-style fold) |
| `Clone` | `(Clone.clone x)` | identity (values are immutable) |

The prelude implements all six for `Int`, `Float`, `Bool`, `String`, `List`, `Option` and `Result`; `Vec` and `Map` implement only `Show`. Every derive checks its fields: a field type without the trait, a `Secret` field under `Eq`/`Ord`/`Hash`, or a trait name that is not derivable is a located `E_TRAIT_NOT_DERIVABLE`.

## Bad

```lisp
(defstruct P (x Int))
(defstruct W (p P))
(derive W Eq)          ; E_TRAIT_NOT_DERIVABLE: field type `P` does not implement `Eq`

(defstruct Login (user String) (pw Secret))
(derive Login Hash)    ; E_TRAIT_NOT_DERIVABLE: it has a Secret field

(derive P Frobnicate)  ; E_TRAIT_NOT_DERIVABLE: `Frobnicate` is not derivable
```

## Good

```lisp
(deftype Shape (Circle Float) (Rect Int Int) (Empty))
(derive Shape Show Eq Ord Hash)
(defstruct Person (name String) (age Int))
(derive Person Show Debug Eq)

(defn main ()
  (begin
    (print (Rect 2 3))                           ; Rect(2, 3)
    (print Empty)                                ; Empty
    (print (make-Person "Ann" 30))               ; Person { name: Ann, age: 30 }
    (print (Debug.debug (make-Person "Ann" 30))) ; Person { name: "Ann", age: 30 }
    (print (Ord.compare (Circle 9.0) (Rect 1 2))); -1: Circle is declared first
    (print (Eq.eq (Rect 1 2) (Rect 1 2)))        ; 1 (a Bool; print shows true as 1)
    0))
```

## Notes

- Derive the field types first (`(derive P Eq)` before or beside `(derive W Eq)`): the check follows each field's declared type.
- **Redaction:** derived `Show`/`Debug` print a `Secret` field (or a field whose type implements the `Secret` trait) as `<secret>`, and a field whose type is protected by `(impl-not Show T)` as `<hidden>`: `(derive Login Show)` with `(pw Secret)` prints `Login { user: ann, pw: <secret> }`. A Secret type and an `impl-not Show` type get a compiler-made `Show` (`<secret>` / `<hidden>`); deriving or writing `Show` for them is `E_IMPL_FORBIDDEN`, and the prelude also declares `(impl-not Debug Secret)`, `(impl-not Eq Secret)`, `(impl-not Ord Secret)` and `(impl-not Hash Secret)` ([trait-coherence-and-orphans](trait-coherence-and-orphans.md)).
- A hand-written `show` whose text derives from a secret (field, binder or helper call) is `E_SECRET_DEBUG`; one that reads an `impl-not Show`-protected field is `E_IMPL_FORBIDDEN`. Print the public fields only, or `declassify`.
- Derive each trait once per type: naming `Show` in two `derive`s (or deriving it beside a hand-written impl) is a located `E_DUPLICATE_IMPL`.
- Works for generic and recursive ADTs: `(StMk "k" (Some 2))` shows `StMk(k, Some(2))`.
- `==` on records is already deep structural without `Eq`. `<`/`>` order only Int, Float and String: on an ADT or struct they are `E_TYPE_MISMATCH` ("ordering on P"), so derive `Ord` and use `Ord.compare` ([data-equality-structural](data-equality-structural.md)).
- `(defstruct+ Name fields... (:derive [Eq Show]))` derives too: it is rewritten into a separate `derive` before qualification, with the same field checks.
- `print` of a struct, or of an Option/Result/List whose payload type has no `Show` impl, prints the raw value (an address). An explicit `(Show.show x)` on a known type with no impl is a located `E_TRAIT_NOT_FOUND` — derive or write the impl; `(Show.show (Some p))` reports it inside `core/option.zyl`, where the payload's `Show.show` is called. There is no run-time fallback ([trait-static-dispatch](trait-static-dispatch.md)).
- The REPL keeps a `derive` entry as a definition.

## See Also

- [trait-static-dispatch](trait-static-dispatch.md)
- [fn-print-semantics](fn-print-semantics.md)
