# err-helper-argument-order

> Option/Result helpers take the value first and the function second; `collections/collections` list helpers take the collection **last**; `-unwrap` helpers require a default.

## Why It Matters

Argument order differs between modules. The type checker catches most swaps (`(option-map (fn (x) x) (Some 1))` is `E_TYPE_MISMATCH: cannot unify (Option ...) with (... -> ...)`), but the message names types, not the mistake, and a swap of two arguments of one type still compiles. Several helpers also differ from the Rust names they resemble: `option-unwrap`/`result-unwrap` take a **default** and never panic.

## Good

```lisp
(option-map (Some 21) (fn (x) (* x 2)))           ; Some(42)
(option-flatmap (Some 2) (fn (x) (option-some (* x 5))))
(result-and-then (Ok 3) (fn (x) (Ok (* x 10))))   ; Ok(30)
(result-unwrap (Err "oops") -1)                    ; -1
(option-expect o "no value")                       ; value, or PANIC: no value

(use collections/collections)
(list-map f xs)          (list-nth n xs)           ; collection last; list-nth gives an Option
(assoc-get k default m)  (assoc-put k v m)
```

## Notes

- Full helper sets: `option-{some,none,is-some,is-none,unwrap,unwrap-or,expect,map,flatmap,and,or,inspect}`, `result-{ok,err,is-ok,is-err,unwrap,unwrap-or,expect,map,flatmap,and-then,or-else,and,or,inspect}`, `option-to-result`, `option-from-result`, `result-to-option`, `result-from-option`.
- Constructors work directly inside lambdas: `(fn (x) (Ok x))` is fine (an old report of it hanging no longer applies).
- `car`/`cdr` return an `Option` (lists are an ADT).
- The book's `list-nth` in the walkthrough is a local helper `(list-nth l k)`; the stdlib one is `(list-nth n xs)`. Defining your own collides if you also `use collections/collections`.

## See Also

- [reference: stdlib](../references/stdlib.md)
- [closure-capture-by-value](closure-capture-by-value.md)
