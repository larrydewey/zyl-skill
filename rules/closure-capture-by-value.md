# closure-capture-by-value

> Expect a closure to see each captured variable's value at the moment the closure was created.

## Why It Matters

Free variables are copied into a heap environment block when the closure is built. Later `set!`s to the original binding are invisible to the closure, and the closure cannot `set!` a capture (`E_MUT_CONFLICT`). A closure built in a loop captures that iteration's value.

## Good

```lisp
(defn make-adder (n) (fn (x) (+ x n)))
(defn main ()
  (let-mut n 5
    (let add (make-adder n)
      (begin
        (set! n 100)
        (print (add 1))      ; 6: the closure holds its own copy
        0))))
```

## Works today

| Shape | Status |
|---|---|
| non-capturing lambda: bind, call, pass, store | works (lifted to a plain function) |
| capturing lambda: call, pass, store, return, capture in another lambda | works |
| lambda calling a captured function value (`compose`, `partial`) | works |
| call through a computed function value | works |
| `set!` on a captured variable | `E_MUT_CONFLICT` |
| recursive lambda | not supported (`E_UNBOUND_VARIABLE`) |
| capturing lambda given to `spawn` | works (immutable captures only; `let-mut` is `E_CAPABILITY_LEAK`) |
| closures of one type in a list, matched out and called | works |
| closure passed to a foreign `ffi-call` | rejected (`E_INVALID_CAPABILITY`); pass a top-level function |

## Notes

- Cost: capturing closure = one allocation for `[tag code env]` plus one for the env; call = indirect call plus a tag test. Prefer non-capturing lambdas and named helpers for anything longer than a line.
- A closure is typed like any function, `(Fn (A ...) R)`: every closure stored together or passed for one parameter must have one type.
- A closure called through a function value makes what it captures escape to the process heap (region inference cannot see the callee).

## See Also

- [own-no-closure-captured-mutation](own-no-closure-captured-mutation.md)
- [fn-types-drive-codegen](fn-types-drive-codegen.md)
- [actor-spawn-zero-arg-entry](actor-spawn-zero-arg-entry.md)
