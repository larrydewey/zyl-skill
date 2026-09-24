# fn-no-named-let-or-early-return

> There is no `return`, named `let`, `let*` or TCO: structure code as small recursive helpers with accumulators, or `while` loops.

## Why It Matters

The last expression is the function's value. Named let and `let*` are not supported. Every call, including tail calls, is a real `call` instruction: deep recursion works only because `main` runs on a very large reserved stack (64 GiB reservation, falling back to 16/4/1 GiB). Actor threads get the default ~8 MB pthread stack, so deep recursion that works in `main` can overflow in an actor.

## Bad

```lisp
(defn process (x)
  (if (< x 0) (return (Err "negative")))   ; no return
  (Ok (* x 2)))
```

## Good

```lisp
(defn process (x)
  (if (< x 0)
    (Err "negative")
    (if (== x 0) (Ok 0) (Ok (* x 2)))))

(defn sum-to (n acc)                        ; accumulator style
  (if (== n 0) acc (sum-to (- n 1) (+ acc n))))
```

## Notes

- Recursion depth in the millions is fine in `main`. For unbounded iteration, prefer `while`.
- Forward references and mutual recursion work: all functions are collected before inference.
- Idiom from the stdlib/compiler: `helper-h` with extra index/accumulator parameters, plus a small public wrapper; reverse an accumulated `List` at the end with `list-reverse`.

## See Also

- [fn-for-has-no-step](fn-for-has-no-step.md) - loops
- [actor-limits](actor-limits.md) - small actor stacks
