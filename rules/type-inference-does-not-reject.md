# type-inference-does-not-reject

> Be your own type checker: the compiler infers types but does not reject ill-typed programs.

## Why It Matters

The Hindley–Milner inferer computes types, but on a failed unification it substitutes a fresh type variable and **continues**. `E_TYPE_MISMATCH`, `E_RETURN_TYPE_MISMATCH`, `E_UNKNOWN_TYPE`, `E_CANNOT_INFER`, `E_TRAIT_BOUND_NOT_SATISFIED` are catalogued and never raised. Annotations are not enforced. These all compile:

```lisp
(defn add ((a Int) (b Int)) (+ a b))
(defn main ()
  (begin
    (print (+ 1 "a"))       ; adds a string's address to 1
    (print (add 1 "x"))     ; annotation not enforced
    (print (+ 1.5 2))       ; prints 1.500000
    0))
```

## What IS checked (before inference)

`E_UNBOUND_VARIABLE`, `E_ARITY_MISMATCH`, `E_DUPLICATE_DEFINITION`/`_VARIANT`/`_PARAMETER`, `E_MALFORMED_PARAMETER`, `E_MUT_CONFLICT`, `E_CAPABILITY_LEAK`, match exhaustiveness/reachability, Secret rules, package capabilities. Inference itself raises only `E_INVALID_CAPABILITY` (non-pinnable FFI value) and the parser `E_BYTE_VALUE_OOB`.

## How to compensate

- Keep Int and Float strictly apart; never pass a String where an Int is expected.
- Name things by type (`n-count`, `s-name`) in weakly-typed helpers.
- Test every function with realistic inputs; mismatched `match` arm types and wrong argument orders only show up at run time.
- Compare `zyl eval` against the compiled binary when behavior is odd.

## Notes

- Inference results **are used**: they choose print formats, feed the FFI pinnability check and record generic ADT instantiations.
- The REPL `:type` reports many applications as *unresolved*: some internal name lookups compare dynamically built strings with `=` (pointer compare).

## See Also

- [type-polymorphic-results-typed-printers](type-polymorphic-results-typed-printers.md)
- [fn-int-float-separation](fn-int-float-separation.md)
