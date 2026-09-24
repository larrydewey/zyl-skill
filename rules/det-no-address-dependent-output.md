# det-no-address-dependent-output

> Never let an address influence output: don't print pointers, don't compare strings or records by address, don't derive names or ordering from pointers.

## Why It Matters

Heap addresses vary per run (ASLR on arena memory) and between compiler stages (allocation order changes). Anything that prints, sorts, hashes or names by address is non-deterministic. In the compiler it breaks the fixed point; in programs it makes output unreproducible. Code addresses are fixed by `-no-pie`.

## Bad

```lisp
(print some-struct)                  ; no Show impl: prints an address
(if (= a b) ...)                     ; a, b of conflicting/unknown type: address compare
```

## Good

```lisp
(derive SomeStruct Show) (print some-struct)
(if (> (str-eq a b) 0) ...)
```

## See Also

- [fn-string-equality](fn-string-equality.md)
- [pass-string-eq-in-compiler](pass-string-eq-in-compiler.md)
