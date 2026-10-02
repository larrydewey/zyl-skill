# err-helper-argument-order

> Option/Result helpers take the value first and the function second; `collections/collections` list helpers take the collection **last**; `-unwrap` helpers require a default.

## Why It Matters

Argument order differs between modules. The type checker catches most swaps (`(option-map (fn (x) x) (Some 1))` is `E_TYPE_MISMATCH`: "`option-map` takes `(Option a)` as its 1st argument (`opt`), but this is `(a -> a)`"), and the message now names the argument, but a swap of two arguments of one type still compiles. Several helpers also differ from the Rust names they resemble: `option-unwrap`/`result-unwrap` take a **default** and never panic; the panicking forms are spelled `option-expect!`/`result-expect!` ([err-partial-ops-spelled](err-partial-ops-spelled.md)).

## Good

```lisp
(use core/option)
(use core/result)
(use collections/collections)

(defn main ()
  (begin
    (print (option-map (Some 21) (fn (x) (* x 2))))                      ; Some(42)
    (print (option-flatmap (Some 2) (fn (x) (option-some (* x 5)))))     ; Some(10)
    (print (result-unwrap (result-and-then (Ok 3) (fn (x) (Ok (* x 10)))) 0))  ; 30
    (print (result-unwrap (Err "oops") -1))                              ; -1
    (print (option-expect! (Some 7) "no value"))                         ; 7; None would panic with "no value"
    (print (list-map (fn (x) (+ x 1)) [1 2 3]))                          ; [2, 3, 4]: collection last
    (print (option-unwrap (list-nth 1 [10 20 30]) -1))                   ; 20: list-nth gives an Option
    (print (assoc-get "b" 0 (assoc-put "b" 2 (assoc-put "a" 1 (assoc-empty)))))    ; 2
    0))
```

## Notes

- Full helper sets: `option-{some,none,is-some,is-none,unwrap,unwrap-or,expect!,map,flatmap,and,or,inspect}`, `result-{ok,err,is-ok,is-err,unwrap,unwrap-or,expect!,map,flatmap,and-then,or-else,and,or,inspect}`, `option-to-result`, `option-from-result`, `result-to-option`, `result-from-option`.
- Constructors work directly inside lambdas: `(fn (x) (Ok x))` is fine (an old report of it hanging no longer applies).
- `car`/`cdr` return an `Option` (lists are an ADT).
- The book's `list-nth` in the walkthrough is a local helper `(list-nth l k)`; the stdlib one is `(list-nth n xs)`. Defining your own collides if you also `use collections/collections`.

## See Also

- [reference: stdlib](../references/stdlib.md)
- [closure-capture-by-value](closure-capture-by-value.md)
