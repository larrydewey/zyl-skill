# fn-numeric-policy

> Int `+ - *` are checked by default: an overflow stops the program with `E_OVERFLOW`. Opt out per file (or in `zyl.pkg`) with `(numeric wrapping)` or `(numeric saturating)`, or per operation with `wrapping+`, `saturating*` and friends.

## Why It Matters

Spec §20: a runtime failure may only happen where the source asked for it, and no wrap-around happens that the author did not choose. Every package has one policy for what Int `+`, `-` and `*` do when the true result does not fit an Int:

| Policy | On overflow |
|---|---|
| checked (the default; `(numeric checked)` says it explicitly) | the program stops: `PANIC: error[E_OVERFLOW]: integer overflow in `+``, catchable with `try` |
| `(numeric wrapping)` | modulo 2^64, as the hardware does |
| `(numeric saturating)` | clamped to the largest or smallest Int, by the sign the true result would have |

Checked arithmetic is enforced in the emitted code: each add, subtract or multiply is followed by a jump on the overflow flag to a runtime trap, one never-taken branch per operation. The optimizer never moves, merges, reassociates or deletes a checked or saturating operation. Code that used to rely on silent wrap-around (hashes, PRNGs, checksums) now stops instead, so spell the wrap.

## Bad

```lisp
(defn fnv-step ((h Int) (b Int)) (* (+ h b) 1099511628211))   ; overflows: E_OVERFLOW at run time
(defn main () (begin (print (fnv-step 9223372036854775806 7)) 0))
```

## Good

```lisp
(defn total ((a Int) (b Int)) (+ a b))
(defn fnv-step ((h Int) (b Int)) (wrapping* (wrapping+ h b) 1099511628211))
(defn clamp-add ((a Int) (b Int)) (saturating+ a b))
(defn main ()
  (begin
    (print (total 2 3))                           ; 5
    (print (fnv-step 9223372036854775806 7))      ; -9223366539296634753
    (print (clamp-add 9223372036854775807 1))     ; 9223372036854775807
    (print (try (total 9223372036854775807 1) (catch _ -1)))   ; -1
    (print (total 9223372036854775807 1))         ; PANIC: error[E_OVERFLOW]: integer overflow in `+`
    0))
```

A whole file whose arithmetic is meant to wrap declares it once:

```lisp
(numeric wrapping)
(defn lcg ((x Int)) (+ (* x 6364136223846793005) 1442695040888963407))
(defn main () (begin (print (lcg 1)) 0))   ; 7806831264735756412
```

## Notes

- The explicit operators are `wrapping+`, `wrapping-`, `wrapping*`, `saturating+`, `saturating-` and `saturating*`. They take Int operands only and ignore the file's policy. The standard library and derived `Hash` impls use them.
- The policy belongs to the package: one `(numeric P)` form among a lone file's top-level forms, or one line in `zyl.pkg`. A `(numeric P)` inside a module file or in the root file of a manifested package, a name other than `checked`, `wrapping` and `saturating`, or two forms that disagree, is `E_MALFORMED_FORM`.
- The standard library, the compiler and the runtime compile checked. The REPL is checked until a `(numeric P)` entry changes it, and the REPL interpreter and the constant folder decide overflow with the same procedure as compiled code (`int_arith.zyl`); an overflowing constant expression is kept as an instruction and traps at run time.
- Unary minus is `(- 0 x)` and follows the policy, so negating the smallest Int is `E_OVERFLOW` when checked.
- Float `+ - * /` are IEEE-754 and outside the policy: overflow is infinity, never an error.
- Division is a separate rule: `/` and `%` need a nonzero literal divisor whatever the policy ([fn-division-total](fn-division-total.md)).
- Known defect (2026-10-02): `print` of the smallest Int, -9223372036854775808, panics with `E_OVERFLOW` in `-` inside the runtime's number formatting, even under `(numeric wrapping)`. Avoid printing that exact value.
- An `E_OVERFLOW` or `E_DIVISION_BY_ZERO` panic currently prints no function-name backtrace, unlike other panics.

## See Also

- [fn-division-total](fn-division-total.md)
- [fn-int-float-separation](fn-int-float-separation.md)
- [syn-int-literal-range](syn-int-literal-range.md)
- [pkg-lone-file-capabilities](pkg-lone-file-capabilities.md)
