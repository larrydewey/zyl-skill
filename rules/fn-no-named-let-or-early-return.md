# fn-no-named-let-or-early-return

> There is no `return`, named `let` or `let*`: structure code as small tail-recursive helpers with accumulators (tail calls, direct or through a function value, are jumps), or `while` loops.

## Why It Matters

The last expression is the function's value. Named let and `let*` are not supported. A call in tail position (`if` branch, `let` body, last `begin` form, `match` arm) is a jump and runs in constant stack, whether it calls a top-level function directly or goes through a function value, provided its stack arguments (those beyond the sixth) fit in the caller's own incoming stack-argument area: a function with eight parameters can tail-call itself or any callee taking up to eight. The exceptions are real `call`s: a tail call needing more stack arguments than the caller received, calls inside `try`/`catch` or `while`, and calls from frame-wiping functions (those holding a `Secret`, which zero their frame on return). Deep recursion through a real call works only because `main` runs on a very large reserved stack (64 GiB reservation, falling back to 16/4/1 GiB). Actor threads get the default ~8 MB pthread stack, so deep non-tail recursion that works in `main` can overflow in an actor.

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

- Recursion depth in the millions is fine in `main`; a tail-recursive loop runs in constant stack anywhere, actors included.
- The REPL interpreter (`zyl repl`, `zyl eval`) also runs tail calls in constant stack, except calls whose result is a String or Float (retagged after the call).
- A body with `ensures` loses its tail calls: the postcondition runs after the call ([contract-not-enforced](contract-not-enforced.md)).
- Forward references and mutual recursion work: all functions are collected before inference.
- Idiom from the stdlib/compiler: `helper-h` with extra index/accumulator parameters, plus a small public wrapper; reverse an accumulated `List` at the end with `list-reverse`.

## See Also

- [fn-for-has-no-step](fn-for-has-no-step.md) - loops
- [actor-limits](actor-limits.md) - small actor stacks
