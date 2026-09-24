# gen-operators-not-overloaded

> Pass comparison and combination functions explicitly in generic code; operators compare machine words.

## Why It Matters

`<` in a polymorphic function compares the words it receives. On strings that is an **address** comparison; on structs/ADTs the code generator applies structural comparison only when it *knows* the operands are variants, which it usually does not inside a polymorphic body. There are no trait bounds to dispatch through.

## Bad

```lisp
(defn smaller (a b) (if (< a b) a b))
(smaller "apple" "banana")               ; compares addresses
```

## Good

```lisp
(defn smaller-by (lt a b) (if (lt a b) a b))
(defn str-lt ((a String) (b String)) ...)  ; your content comparison
(smaller-by str-lt "apple" "banana")
(smaller-by (fn (x y) (< x y)) 3 5)
```

## See Also

- [fn-string-equality](fn-string-equality.md)
- [data-equality-shallow](data-equality-shallow.md)
