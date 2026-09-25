# data-struct-basics

> Declare with `defstruct`, build with `make-Name`, read with `v.field` (chains: `v.a.b`; any expression: `(expr).field`) or `(struct-get v "field")` with a literal field name.

## Why It Matters

`v.field` is rewritten to `(struct-get v "field")` before qualification (first segment lowercase, after any leading `_`), and so is `(expr).field` on any expression's value: `(make-P 1 2).x`, `(seg).b.y`, `(Nm.nm p).x`. The field name is fixed at compile time: `struct-get` takes a string literal (a bare identifier is also read as the field's name, not as a variable). A field the struct lacks is `E_TYPE_MISMATCH: no field `z` on struct `Person``. A field read whose struct type is still undetermined after inference, when more than one struct has that field, is `E_CANNOT_INFER` (annotate the parameter). A struct is a single-variant ADT named after itself, so `(Point 1 2)` builds the same value as `(make-Point 1 2)` and a struct can be `match`ed with one arm.

## Bad

```lisp
(let key "x" (struct-get p key))      ; field named `key`: no field `key` on struct `Point`
(struct-get p (str-concat "" "x"))    ; E_TYPE_MISMATCH: no field `` (no computed keys)
(struct-get p 'x)                     ; quoted data, not a name: no field ``
(defstruct A (x Int)) (defstruct B (x Int))
(defn getx (r) r.x)                   ; E_CANNOT_INFER: more than one struct has `x`
```

## Good

```lisp
(defstruct Point x y)                      ; bare field names (generic)
(defstruct Size (w) (h))                   ; parenthesized (generic)
(defstruct Person (name String) (age Int)) ; typed

(defn main ()
  (let p (make-Point 3 4)
    (let alice (make-Person "Alice" 30)
      (begin
        (print (struct-get p "x"))                 ; 3
        (print alice.name)                         ; Alice
        (print (make-Point 1 2).y)                 ; 2
        (print (match p (Point x y (+ x y))))      ; 7
        0))))
```

## Notes

- Constructor arguments are evaluated left to right and matched to fields in declaration order; the arity and the field types are checked (`(make-Person 30 "Ann")` is `E_TYPE_MISMATCH`, pass `true`/`false` to Bool fields).
- Untyped fields are type parameters of the struct ([data-no-tuples-implicit-generic-structs](data-no-tuples-implicit-generic-structs.md)).
- A trait call works as `struct-get`'s first argument or before `.field`: `(struct-get (Nm.nm p) "x")`, `(Nm.nm p).x`. In head position `((expr).f.m args)` reads the fields, then calls method `m` on the result ([trait-qualified-calls](trait-qualified-calls.md)).
- `defstruct+` defines exactly the same struct; its `(:derive [Eq Show])` clause becomes a separate `derive` ([trait-derive-show](trait-derive-show.md)).
- Each struct gets a program-unique tag (from 100000 upward).

## See Also

- [data-struct-immutable-rebind](data-struct-immutable-rebind.md)
- [data-field-types](data-field-types.md) - printing string/float fields
