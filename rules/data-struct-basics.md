# data-struct-basics

> Declare with `defstruct`, build with `make-Name`, read with `(struct-get v "field")` — the field name is a string.

## Why It Matters

`struct-get` takes a **string literal** field key, not a symbol. A struct is a single-variant ADT named after itself, so `(Point 1 2)` builds the same value as `(make-Point 1 2)` and a struct can be `match`ed with one arm. Each struct gets a program-unique tag (from 100000 upward), which is what `struct-get` and trait dispatch rely on.

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
(defstruct Person (name String) (age Int)) ; typed (types are documentation)

(defn main ()
  (let p (make-Point 3 4)
    (let alice (make-Person "Alice" 30)
      (begin
        (print (struct-get p "x"))                 ; 3
        (print-string (struct-get alice "name"))   ; Alice
        (print (match p (Point x y (+ x y))))      ; 7
        0))))
```

## Notes

- Constructor arguments are evaluated left to right and matched to fields in declaration order.
- `struct-get` lowers to a `match` with one arm per struct type that has that field.
- A trait call written directly as the first argument of `struct-get` is not rewritten and fails to link; bind it first (see [trait-bind-before-struct-get](trait-bind-before-struct-get.md)).
- `defstruct+` defines exactly the same struct (its `:derive` clause is a no-op).
- Field type annotations are dropped in lowering; they are not checked.
- Generic structs are not supported; use a generic ADT.

## See Also

- [data-struct-immutable-rebind](data-struct-immutable-rebind.md)
- [data-field-kinds](data-field-kinds.md) - printing string/float fields
