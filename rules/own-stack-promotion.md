# own-stack-promotion

> Keep short-lived values from escaping and the compiler places them in the call's frame region; a `let`-bound ADT that is only matched or printed goes further, into the frame itself.

## Why It Matters

Two mechanisms keep values off the process heap, both automatic:

1. **Frame region (per-call regions).** Any allocation whose object class does not reach the result or escape is placed in the call's own region and released on return. Passing the value to a compiled function is fine when that function's summary says the parameter does not escape.
2. **`IStackVariant`.** For exactly one shape, `(let x (Variant ...) body)` where every use of `x` is a `match` scrutinee or a `print` argument and no nested `fn` references it, the variant is stored directly in the frame (no allocation call at all).

What forces the heap: storing into a global, `send`, a foreign `ffi-call`, capture by a closure called through a function value, or a runtime function not in the compiler's region-safe table. Returning a value puts it in the caller's region, not the heap.

## Good

```lisp
(deftype Shape (Circle Int) (Square Int))
(defn area (s) (match s (Circle r (* 3 (* r r))) (Square n (* n n))))
(defn local-area ()
  (let s (Square 4)              ; IStackVariant: only matched
    (match s
      (Circle r r)
      (Square n (* n n)))))

(defn passed-area ()
  (let s (Square 4)
    (area s)))                   ; frame region: area does not keep s
```

## Heap (still correct)

```lisp
(defn stash (target)
  (send target (Square 4)))      ; sent: heap
```

## Notes

- Both are optimizations; a missed case costs memory, never correctness. For bulk temporaries with an explicit bound, use [own-with-region](own-with-region.md).
- Small functions are inlined before region inference (`optimization.zyl`, `ZYL_INLINE=0` to disable, `ZYL_INLINE_LIMIT` for the size), so a helper's allocation often becomes a local one of its caller.
- A third mechanism avoids allocation altogether: the reuse pass writes an update of a unique, dead value into its old block ([own-heap-never-freed](own-heap-never-freed.md)).

## See Also

- [own-heap-never-freed](own-heap-never-freed.md)
- [icnf-regions-are-a-rewrite](icnf-regions-are-a-rewrite.md)
