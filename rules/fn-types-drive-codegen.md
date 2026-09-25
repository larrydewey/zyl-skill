# fn-types-drive-codegen

> Let inference type your values: `print`, `=`/`<` and arithmetic follow the static type the checker proved for every expression; a program whose types conflict or cannot be determined does not compile, so there is no untyped fallback to guard against.

## Why It Matters

Code generation picks `%lld`/`%f`/`%s`, String comparison (`zyl_cstr_eq`, `zyl_cstr_cmp`), SSE Float arithmetic and `Show`-based printing from a static type. That type comes from `compiler/type_annotate.zyl`, a sound Hindley–Milner pass over the whole program. Since 2026-09-25 it is strict: every type error is reported, then the compile fails. So all of these work without annotations:

```lisp
(deftype Shape (Circle Float) (Rect Float Float))
(defn area (s) (match s (Circle r (* 3.14 (* r r))) (Rect w h (* w h))))
(defn half (x) (/ x 2.0))
(defn shout (s) (print (str-concat s "!")))

(defn main ()
  (let who "bob"
    (let f (fn (n) (str-concat who n))
      (begin
        (print (area (Rect 1.5 2.0)))   ; 3.000000
        (print (half 3.0))              ; 1.500000
        (print (f "!"))                 ; bob!
        (print (= (str-concat "a" "b") "ab"))  ; 1
        (shout "hi")                    ; hi!
        0))))
```

## What used to fall back, and what happens now

| Situation | Before 2026-09-25 | Now |
|---|---|---|
| A variable used as both Int and String | printed/compared as a raw word | `E_TYPE_MISMATCH`, located |
| A list mixing element types | raw words | `E_TYPE_MISMATCH` at the literal |
| An `ffi-call` to a foreign symbol with no `extern` | untyped word | `E_CANNOT_INFER` ("which has no (extern ...) declaration") |
| A runtime `zyl_*` entry missing from `ffi_sigs.zyl` | untyped word | `E_CANNOT_INFER` ("untyped ffi result") |
| A trait call on an unresolved receiver | run-time tag dispatch | `E_CANNOT_INFER`; calls resolve statically |
| `vec-get` out of range | a `-1` sentinel | panic `E_INDEX_OUT_OF_BOUNDS` |

The one known hole is `receive`, whose result is any type.

## Notes

- Inside a generic body, type-dependent operations are handled by per-type instances, and a trait-generic function used as a value is specialized too: [gen-per-type-instances](gen-per-type-instances.md).
- `ZYL_DEBUG_TYPES=1 zyl file.zyl` prints every function's inferred type (`half : ( Float -> Float)`); the REPL's `:type expr` shows one expression's. `ZYL_STRICT_TYPES=report` turns type errors into `W_TYPE_STRICT` warnings for counting only.
- Type errors reach the editor: the language server runs the same checker and publishes every located type error.
- A function whose operators all work on Int-like values can be compiled by the register-allocating native backend; String, Float or ADT operands keep that function on the stack-machine backend. Output is the same either way.
- Bool still prints `1`/`0`.

## See Also

- [type-sound-checking](type-sound-checking.md)
- [data-field-types](data-field-types.md)
- [cg-kind-of](cg-kind-of.md) - the mechanism
