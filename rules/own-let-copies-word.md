# own-let-copies-word

> Know that every value is one 64-bit word and `let` copies the word: scalars become independent copies, pointers become aliases.

## Why It Matters

Binding a second name to a `let-mut` Int gives an independent copy, so later `set!`s do not affect it. For structs, ADTs, strings and collections the word is a **pointer**, so two names refer to the same block. That is safe for structs/ADTs/strings because they are immutable — and unsafe for `Vec`/`Map`/`Set`, whose operations write into the shared buffer.

## Good

```lisp
(defn main ()
  (let-mut x 1
    (let y x
      (begin
        (set! x 2)
        (print y)     ; 1: y is a copy
        (print x)     ; 2
        0))))
```

## Bad

```lisp
(use collections/map)
(use collections/vec)
(let m1 (map-put (map-create-default 4) 1 100)
  (let m2 (map-put m1 1 111)      ; overwrites in the shared buffer
    (map-get m1 1 0)))            ; 111, not 100

(let v1 (vec-push (vec-create-default 4) 1)
  (let v2 (vec-push v1 2)         ; writes slot 1 of the shared storage
    (let v3 (vec-push v1 3)       ; writes slot 1 again
      (vec-get v2 1))))           ; 3, not 2
```

## Notes

- Representation: Int untagged; Float as raw IEEE bits; Bool 0/1; String = pointer to NUL-terminated bytes; struct/ADT = pointer to `[hidden size][tag][field0]...`; capturing closure = pointer to `[tag code env]`; Vec/Map = ordinary structs.
- Capability and region information is compile-time only and erased.
- The reuse pass (`compiler/reuse.zyl`) writes an update into the old block only when no second name can see it, so it never causes this aliasing; the collections' shared storage does.

## See Also

- [data-collections-persistent](data-collections-persistent.md)
- [type-value-representation](type-value-representation.md)
