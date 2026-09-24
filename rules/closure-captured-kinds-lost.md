# closure-captured-kinds-lost

> Don't `print`, `=`-compare or do float arithmetic directly on captured variables or on results of calls through function values; pass them to typed functions.

## Why It Matters

Code generation picks string/float handling from annotations and literals only. A captured variable, an unannotated parameter and the result of an indirect call are all treated as integers there. `(let s "hi" (let g (fn () (print s)) (g)))` prints the string's **address**; a captured Float is added as an integer.

## Bad

```lisp
(let s "hi" (let g (fn () (print s)) (g)))            ; address
(let r 2.0 (let area (fn () (* 3.14 r r)) (area)))    ; garbage
```

## Good

```lisp
(let s "hi" (let g (fn () (print-string s)) (g)))     ; hi
(defn circle ((r Float)) (* 3.14 r r))
(let r 2.0 (let area (fn () (circle r)) (print-float (area))))
```

## See Also

- [fn-annotate-string-float-params](fn-annotate-string-float-params.md)
- [cg-kind-of](cg-kind-of.md)
