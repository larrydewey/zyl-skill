# macro-name-positions

> Pass an identifier when a macro parameter lands in a name position (binder, `set!` target, `defn`/`deftype`/`impl` name).

## Why It Matters

Parameters are substituted in name positions too, which is how macros can define functions or bind caller-named variables. A non-identifier argument in such a position is `E_MALFORMED_PARAMETER`.

## Good

```lisp
(defmacro bind (n v body) (let n v body))
(print (bind q 7 (+ q 1)))                 ; 8

(defmacro def-square (name) (defn name (k) (* k k)))
(def-square sq)
(print (sq 5))                             ; 25
```

## Bad

```lisp
(bind (+ 1 2) 7 0)                         ; E_MALFORMED_PARAMETER
```

## See Also

- [macro-hygiene](macro-hygiene.md)
