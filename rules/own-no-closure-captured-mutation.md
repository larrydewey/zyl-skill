# own-no-closure-captured-mutation

> Never `set!` a captured variable inside a closure; have the closure return a new value and rebind it in the owning scope.

## Why It Matters

Closures capture **by value**: when the closure is created, each captured variable's current value is copied into a heap environment. A `set!` inside the closure could only change the copy. The current compiler rejects it with a located `E_MUT_CONFLICT` ("closures capture by value (spec 7); return the new value from the closure and set! it at the binding's own scope"). Older builds accepted it and segfaulted at run time (Chapter 17 still describes that), so treat it as forbidden regardless of compiler version.

## Bad

```lisp
(defn main ()
  (let-mut count 0
    (let bump (fn () (set! count (+ count 1)))   ; E_MUT_CONFLICT
      (begin (bump) (print count) 0))))
```

## Good

```lisp
(defn main ()
  (let-mut count 0
    (let inc (fn (c) (+ c 1))
      (begin
        (set! count (inc count))
        (set! count (inc count))
        (print count)          ; 2
        0))))
```

## Notes

- A closure may freely `set!` its **own** `let-mut` locals.
- For state shared between closures, pass a data structure explicitly; across actors, use `atomic/atomic` or `bytebuf-atomic-*` on shared memory.

## See Also

- [closure-capture-by-value](closure-capture-by-value.md)
- [actor-no-let-mut-crossing](actor-no-let-mut-crossing.md)
