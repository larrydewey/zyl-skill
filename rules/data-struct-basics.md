# data-struct-basics

> Declare with `defstruct`, build with `make-Name`, read with `v.field` (chains: `v.a.b`) or `(struct-get v "field")`.

## Why It Matters

`v.field` is rewritten to `(struct-get v "field")` before qualification (first segment lowercase, after any leading `_`); `struct-get` itself takes a **string literal** field key, not a symbol. A field a known struct lacks is `E_TYPE_MISMATCH: no field ...` in either spelling. A struct is a single-variant ADT named after itself, so `(Point 1 2)` builds the same value as `(make-Point 1 2)` and a struct can be `match`ed with one arm. Each struct gets a program-unique tag (from 100000 upward), which is what `struct-get`'s lowering and the runtime fallback of trait dispatch rely on.

## Bad

```lisp
(struct-get p x)          ; x is a variable reference, not the field name
(struct-get p 'x)         ; quote truncates the file
p.x                       ; one identifier named "p.x"
```

## Good

```lisp
(defstruct Point x y)                      ; bare field names
(defstruct Size (w) (h))                   ; parenthesized
(defstruct Person (name String) (age Int)) ; typed (constructor args checked)

(defn main ()
  (let p (make-Point 3 4)
    (let alice (make-Person "Alice" 30)
      (begin
        (print (struct-get p "x"))                 ; 3
        (print (struct-get alice "name"))          ; Alice (declared String field)
        (print (match p (Point x y (+ x y))))      ; 7
        0))))
```

## Notes

- Constructor arguments are evaluated left to right and matched to fields in declaration order.
- `struct-get` lowers to a `match` with one arm per struct type that has that field.
- A trait call written directly as the first argument of `struct-get` is not rewritten and fails to link; bind it first (see [trait-qualified-calls](trait-qualified-calls.md)).
- `defstruct+` defines exactly the same struct (its `:derive` clause is a no-op).
- Field type annotations are checked at constructor calls: `(make-Person 30 "Ann")` is `E_TYPE_MISMATCH` (definite clashes only; pass `true`/`false` to Bool fields). Arity is not checked, and the types are dropped in lowering.
- Generic structs are not supported; use a generic ADT.

## See Also

- [data-struct-immutable-rebind](data-struct-immutable-rebind.md)
- [data-field-types](data-field-types.md) - printing string/float fields
