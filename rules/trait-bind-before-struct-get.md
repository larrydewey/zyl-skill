# trait-bind-before-struct-get

> Bind a trait call's result with `let` before passing it to `struct-get`.

## Why It Matters

Compiler defect: a trait call written directly as the first argument of `struct-get` is not rewritten by trait dispatch, and the program fails to link with `undefined reference to _ZYL_Trait_method`.

## Bad

```lisp
(struct-get (Scale.scale r 3) "h")
```

## Good

```lisp
(let r2 (Scale.scale r 3)
  (struct-get r2 "h"))
```

## See Also

- [trait-qualified-calls](trait-qualified-calls.md)
