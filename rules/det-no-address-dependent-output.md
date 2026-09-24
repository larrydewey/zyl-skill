# det-no-address-dependent-output

> Never let an address influence output: don't print pointers, don't compare strings or records by address, don't derive names or ordering from pointers.

## Why It Matters

Heap addresses vary per run (ASLR on arena memory) and between compiler stages (allocation order changes). Anything that prints, sorts, hashes or names by address is non-deterministic. In the compiler it breaks the fixed point; in programs it makes output unreproducible. Code addresses are fixed by `-no-pie`.

## Bad

```lisp
(print some-struct)                  ; prints an address
(print (ident "s"))                  ; address, via a polymorphic call
(if (= name1 name2) ...)             ; unannotated strings: address compare
```

## Good

```lisp
(print-string (ident "s"))
(if (> (str-eq name1 name2) 0) ...)
```

## See Also

- [fn-string-equality](fn-string-equality.md)
- [pass-string-eq-in-compiler](pass-string-eq-in-compiler.md)
