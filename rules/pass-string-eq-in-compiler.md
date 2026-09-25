# pass-string-eq-in-compiler

> In compiler source, compare strings with `str-eq` (a `Bool`) or `=` on String-typed operands; both compare contents. Never compare strings by address.

## Why It Matters

`=` on strings compares contents when codegen knows an operand is a String (`kind-of` 1, then `zyl_cstr_eq`). Before sound typing, dynamically built strings in untyped positions had kind 0 and `=` compared **pointers**: lookups silently missed, and results depended on allocation order, which differs between stage 2 and stage 3 — a fixed-point breaker. Since every expression now has a known type, a String operand always has kind 1, and `=` inside a generic function is specialized per type. Verified 2026-09-25 in both backends: `(= a b)` on two separately built `"abc"`s, the same through a generic `(defn eqp (a b) (= a b))`, and an element search `(if (= h x) ...)` over a `(List String)` all give true.

The compiler source still uses `str-eq` almost everywhere (it predates the fix and reads unambiguously), and it is a `Bool` predicate: write `(if (str-eq a b) ...)`, not `(if (> (str-eq a b) 0) ...)`, which is now a type error.

## Bad

```lisp
(if (> (str-eq n name) 0) ...)                          ; E_TYPE_MISMATCH: str-eq is Bool
(ffi-call "zyl_word_of_cstr" s 1000)                    ; comparing addresses: never
```

## Good

```lisp
(if (str-eq n name) ...)
(if (= n name) ...)                                     ; n, name : String
(ffi-call "zyl_cstr_key_matches" key name 1000)         ; qualified-key match, allocation-free
```

## Notes

- An ordering of strings uses `zyl_cstr_cmp` (bytes), the same for `<` on String operands.
- A native-backend function never compares strings itself: a String operator keeps the function on the stack machine ([cg-kind-of](cg-kind-of.md)).

## See Also

- [fn-string-equality](fn-string-equality.md)
- [det-no-address-dependent-output](det-no-address-dependent-output.md)
