# type-sound-checking

> Write well-typed code: type checking is sound and strict, so every type error in the program is reported (`E_TYPE_MISMATCH`, `E_INFINITE_TYPE`, `E_CANNOT_INFER`, `E_UNBOUND_VARIABLE`) and then the compile fails.

## Why It Matters

The checker used to fail open; it no longer does. `compiler/type_annotate.zyl` is sound Hindley–Milner (spec §4.8–§4.10, `docs/sound-types-design.md`): a program the compiler accepts never uses a value at the wrong representation. There is no cast and no escape hatch, so a type error must be fixed, not worked around. Each failure is reported at the innermost expression being typed, all of them are listed, and the compile then fails with the first error's code.

```lisp
(defn add ((a Int) (b Int)) (+ a b))
(deftype Shape (Circle Int))
(defn main ()
  (begin
    (print (add 1.5 2.0))       ; E_TYPE_MISMATCH: parameter `a` of `add` is declared `Int`
    (print (Circle 1.5))        ; E_TYPE_MISMATCH: field 1 of `Circle` is declared `Int`
    (print (+ 1 "a"))           ; E_TYPE_MISMATCH: cannot unify String with Int
    (print (+ 1.5 2))           ; E_TYPE_MISMATCH: no mixing Int and Float
    ((fn ((a Int)) a) "x")      ; E_TYPE_MISMATCH: lambda parameters are checked too
    (if 1 (print 2) (print 3))  ; E_TYPE_MISMATCH: a condition is Bool
    0))
```

## The rules you will meet

| Rule | Consequence |
|---|---|
| Conditions (`if`, `cond`, `when`, `while`, guards, `assert*`, contract clauses) are `Bool` | `(if 1 ...)`, `(assert 1 "x")`, `(requires 1)` are errors; use `true`/`false` and comparisons |
| `+ - * / %` take two Ints or two Floats | no implicit conversion; `(+ 1.5 2)` is an error |
| `< > <= >=` take Int, Float or String | an ADT or struct operand is `E_TYPE_MISMATCH: ordering on ...`; use `Ord.compare` |
| Statement forms (`print`, `set!`, `while`, `for`, `assert*`, `send`, `(begin)`, `if` without else) are `Unit`; the literal `unit` is its value | `(print (if true 1))` is an error: the missing else is Unit |
| `main` is `() -> Int` | `(defn main () (print 1))` is an error; end `main` with `0` |
| Every `match` arm and both `if` branches have one type | `(match l (Red 1) (Green "g"))` is an error |
| Local `let` is monomorphic; top-level functions generalize; top-level `def` does not | `(let id (fn (x) x) ... (id 1) (id "a"))` is an error |
| `(x x)` | `E_INFINITE_TYPE` (occurs check) |
| A field read whose struct is not determined, a trait call on an unresolved receiver, an `ffi-call` to a foreign symbol with no `extern` | `E_CANNOT_INFER` |
| A name defined nowhere | `E_UNBOUND_VARIABLE` |
| A trait call on a concrete type with no impl | `E_TRAIT_NOT_FOUND` |

Also checked before or during typing: `E_ARITY_MISMATCH` (constructors included), `E_MALFORMED_PARAMETER` (`(a : Int)`, a trait used as a type), `E_MALFORMED_FORM` (a special form of the wrong shape), `E_NESTED_PATTERN`, `E_FFI_RESTRICTED` (raw runtime entries in user code), and the duplicate, mutability, capability, exhaustiveness, Secret and package-capability checks.

## How to work with it

- Read every error, not only the first: one mistake usually shows up as a located `mismatched types: expected ..., found ...` plus a `cannot unify A with B` at the enclosing expression.
- `ZYL_DEBUG_TYPES=1` prints each function's inferred type (`'1`, `'2` are type variables) and each specialized instance; REPL `:type expr` shows one expression's type.
- `ZYL_STRICT_TYPES=report` prints the same diagnostics as `W_TYPE_STRICT` warnings and continues. It exists for counting remaining errors during a port; never ship a program built that way.
- The language server runs the same checker and publishes every type error.

## Known holes

- `receive` returns a value of any type (a mailbox holds whatever senders put there).
- A lowercase name as a `deftype` field type, `(deftype Box (Bx a))`, is `E_UNKNOWN_TYPE` (since 2026-09-25; it used to leave the field unchecked). Write type parameters uppercase ([gen-generic-adts](gen-generic-adts.md)).
- Contract clauses compiled out by the `off`/`production` profile are not type-checked at all.

## Notes

- `E_RETURN_TYPE_MISMATCH`, `E_UNKNOWN_TYPE` and `E_TRAIT_BOUND_NOT_SATISFIED` are catalogued but never raised: there are no return-type annotations or trait bounds to violate, and an unknown uppercase annotation name is a type variable.
- Inference results drive code generation: print formats, String comparison, Float arithmetic, static trait resolution, the generated `T.==`, and per-type instances ([gen-per-type-instances](gen-per-type-instances.md)).

## See Also

- [type-annotations-constrain](type-annotations-constrain.md)
- [gen-per-type-instances](gen-per-type-instances.md)
- [fn-int-float-separation](fn-int-float-separation.md)
