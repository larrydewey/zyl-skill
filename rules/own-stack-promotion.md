# own-stack-promotion

> To keep a short-lived ADT value off the heap, bind it with `let` and only `match` on it or `print` it in the body.

## Why It Matters

Region inference proves non-escape for exactly one shape: `(let x (Variant ...) body)` where every use of `x` in `body` is a `match` scrutinee or a `print` argument, and `x` is not referenced inside a nested `fn`. That construction becomes an `IStackVariant` in the current frame (no `zyl_heap_alloc` call in the assembly). Passing `x` to any function, storing it, returning it or capturing it keeps it on the heap.

## Good

```lisp
(deftype Shape (Circle Int) (Square Int))
(defn local-area ()
  (let s (Square 4)              ; stack: only matched
    (match s
      (Circle r r)
      (Square n (* n n)))))
```

## Heap (still correct)

```lisp
(defn passed-area ()
  (let s (Square 4)
    (area s)))                   ; passed to a function: heap
```

## Notes

- This is an optimization only. It runs on ICNF after optimization, not in the spec's Phase 4. There is no syntax for choosing a region; let the compiler decide.

## See Also

- [own-heap-never-freed](own-heap-never-freed.md)
- [icnf-regions-are-a-rewrite](icnf-regions-are-a-rewrite.md)
