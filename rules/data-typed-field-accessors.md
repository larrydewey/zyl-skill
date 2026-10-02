# data-typed-field-accessors

> Read a struct field with its generated accessor `(Point.x p)`: every `defstruct` defines one typed function `T.f` per field, usable anywhere a function is, and a program `defn` of the same name is `E_DUPLICATE_DEFINITION`.

## Why It Matters

`(defstruct Point (x Int) (y Int))` generates `(defn Point.x ((p Point)) (struct-get p "x"))` and `Point.y` before type checking, visible wherever the struct is. Because the parameter is annotated, a wrong argument is reported at the call with both types (`` `Point.x` takes `Point` as its 1st argument (`p`), but this is `Label` ``), where `struct-get` or `p.x` on an ambiguous field name can only say that the field is ambiguous. An accessor is also an ordinary function value, so it can be passed to `list-map` or a fold without writing a lambda. The language server resolves hover, go-to-definition and completion after `T.` to the field.

## Bad

```lisp
(defstruct Point (x Int) (y Int))
(defn Point.x (p) 7)   ; E_DUPLICATE_DEFINITION: already defined by (defstruct Point ...)
```

## Good

```lisp
(use collections/collections)
(defstruct Point (x Int) (y Int))
(defn norm1 ((p Point)) (+ (Point.x p) (Point.y p)))
(defn xs (ps) (list-map Point.x ps))
(defn main ()
  (let p (make-Point 3 4)
    (begin
      (print (norm1 p))                    ; 7
      (print (xs [p (make-Point 9 0)]))    ; [3, 9]
      (print p.y)                          ; 4: dot access still works
      0)))
```

## Notes

- The duplicate-definition report lands on the program's own `defn` and says the name comes from the `defstruct`; rename the helper (`get-x`) or delete it.
- Accessors read only. Fields stay immutable: build a new value to change one ([data-struct-immutable-rebind](data-struct-immutable-rebind.md)).
- `p.x` and `(struct-get p "x")` remain valid; prefer `T.f` when two structs share a field name.

## See Also

- [data-struct-basics](data-struct-basics.md)
- [data-field-types](data-field-types.md)
- [data-struct-immutable-rebind](data-struct-immutable-rebind.md)
