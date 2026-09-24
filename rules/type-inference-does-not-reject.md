# type-inference-does-not-reject

> Be your own type checker: the compiler rejects only definite clashes with a parameter or field annotation at a call; every other ill-typed program compiles.

## Why It Matters

The Hindley–Milner pass (`compiler/type_annotate.zyl`) computes types, but a failed unification only marks the variables involved as unknown and **continues**. The one check: a call to a top-level function, or a constructor call (`(Circle ...)`, `make-Point`), whose argument's inferred type **definitely** clashes with the declared parameter/field type raises located `E_TYPE_MISMATCH` (structural: `(List String)` vs `(List Int)` is caught). A type variable or unknown part never clashes, nor does `Unit`. `E_RETURN_TYPE_MISMATCH`, `E_UNKNOWN_TYPE`, `E_CANNOT_INFER`, `E_TRAIT_BOUND_NOT_SATISFIED` are catalogued and never raised.

```lisp
(defn add ((a Int) (b Int)) (+ a b))
(deftype Shape (Circle Int))
(defn main ()
  (begin
    (print (add 1.5 2.0))   ; E_TYPE_MISMATCH at 1.5, labelled at `(a Int)`
    (print (Circle 1.5))    ; E_TYPE_MISMATCH: field 1 of `Circle` is `Int`
    (print (+ 1 "a"))       ; compiles: adds a string's address to 1
    (print (+ 1.5 2))       ; compiles: Int and Float mixed
    ((fn ((a Int)) a) "x")  ; compiles: lambda params not checked
    0))
```

## What IS checked (before inference)

`E_UNBOUND_VARIABLE`, `E_ARITY_MISMATCH`, `E_DUPLICATE_DEFINITION`/`_VARIANT`/`_PARAMETER`, `E_MALFORMED_PARAMETER`, `E_MUT_CONFLICT`, `E_CAPABILITY_LEAK`, match exhaustiveness/reachability, Secret rules, package capabilities. `E_INVALID_CAPABILITY` (a lambda passed to `ffi-call`) comes from `mutability_check`, `E_BYTE_VALUE_OOB` from the parser.

## How to compensate

- Keep Int and Float strictly apart in arithmetic; operators are never checked.
- Pass `true`/`false`, not `1`/`0`, to `Bool` params and fields; don't store a String through an Int-annotated helper (e.g. math/words' `w-set`).
- Constructor calls are not arity-checked, trait-method calls and lambdas not type-checked, and unknown annotation names (`(v Bogus)`) are type variables.
- Test every function with realistic inputs; mismatched `match` arm types and wrong argument orders only show up at run time.
- Compare `zyl eval` against the compiled binary when behavior is odd.

## Notes

- The type pass also reports a trait call on a receiver of known type with no impl (`E_TRAIT_NOT_FOUND`, located) and derives whose field types lack the trait (`E_TRAIT_NOT_DERIVABLE`, in `derive.zyl`); neither is a general type check.
- Inference results **are used**: they choose print formats, String comparison and Float arithmetic, resolve trait calls, generate ADT equality and drive per-type instances. A conflict silently degrades those to word semantics for the values involved.
- `ZYL_DEBUG_TYPES=1` prints each function's inferred type; REPL `:type expr` shows one expression's (`a` = unconstrained, `?` = conflicting).

## See Also

- [gen-per-type-instances](gen-per-type-instances.md)
- [fn-int-float-separation](fn-int-float-separation.md)
- [type-annotations-guide-codegen](type-annotations-guide-codegen.md)
