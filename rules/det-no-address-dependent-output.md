# det-no-address-dependent-output

> Never let an address influence output: give every printed type a `Show`, don't compare handles expecting content, don't derive names or ordering from pointers.

## Why It Matters

Heap addresses vary per run (ASLR on arena memory) and between compiler stages (allocation order changes). Anything that prints, sorts, hashes or names by address is non-deterministic. In the compiler it breaks the fixed point; in programs it makes output unreproducible. Code addresses are fixed by `-no-pie`.

Sound typing closed most of the old holes: `=` and `==` on records and ADTs compare by content (the generated structural equality), `=` on Strings compares content, and `<` on an ADT is a type error (use `Ord.compare`), so nothing is ordered by address any more. What remains:

- `print` of a struct or ADT with no `Show` impl prints its address, in compiled code and in `zyl eval` alike.
- `=` on runtime handles (`Words`, `Arena`, `ByteBuf`, ...) compares the handles.
- A record type with a `Secret` field has no generated equality; `=` on it falls back to a shallow word comparison, so its String fields compare by address.

## Bad

```lisp
(defstruct P (x Int) (y String))
(print (make-P 1 "a"))               ; no Show impl: prints an address such as 140123932982304
(= words-a words-b)                  ; two Words arrays: handle comparison, not content
```

## Good

```lisp
(defstruct P (x Int) (y String))
(derive P Show)
(print (make-P 1 "a"))               ; P { x: 1, y: a }
(ct-eq-words-bool words-a words-b n) ; content, in constant time (math/secret/secret)
```

## See Also

- [fn-string-equality](fn-string-equality.md)
- [pass-string-eq-in-compiler](pass-string-eq-in-compiler.md)
