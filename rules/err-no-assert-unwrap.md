# err-no-assert-unwrap

> Never use `assert` or `unwrap` in compiled code: `assert` checks nothing and `unwrap` evaluates to 0.

## Why It Matters

Both parse; neither is implemented by the code generator. `(assert c "msg")` does nothing whatever `c` is (`E_ASSERT_FAIL` is never raised). `(unwrap x)` evaluates to **0**, silently replacing the value you meant to extract.

## Bad

```lisp
(assert (> n 0) "n must be positive")
(let v (unwrap (lookup k)) (* v 2))      ; v is 0
```

## Good

```lisp
(if (<= n 0) (error "n must be positive") 0)
(let v (option-expect (lookup k) "missing key") (* v 2))
(let v (option-unwrap (lookup k) -1) ...)   ; with a default
```

## Notes

- In tests, `assert-true`, `assert-false` and `assert-equal` **do** work (outside a test they panic and exit 1 on failure, so `assert-true` is also a usable runtime check).
- `assert-fail` evaluates its expression and always passes.

## See Also

- [fn-unlowered-forms](fn-unlowered-forms.md)
- [test-assert-equal-semantics](test-assert-equal-semantics.md)
