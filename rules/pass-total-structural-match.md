# pass-total-structural-match

> In tree-rewriting passes, list every constructor explicitly; use `_` only in read-only checks that care about a few forms.

## Why It Matters

A rewriter with a catch-all silently passes a newly added constructor through unchanged (or, worse, rewrites it to a default). Listing every constructor makes a new constructor fail exhaustiveness in every rewriter until handled. Read-only checks can use `_` for the rest.

## Good

```lisp
(defn opt-expr (e)
  (match e
    (IConst _ e)
    (IBinop op l r (opt-binop op (opt-expr l) (opt-expr r)))
    (IIf c t eb (opt-if (opt-expr c) t eb))
    (ILet name val body (ILet name (opt-expr val) (opt-expr body)))
    ...))                    ; one arm per Icnf constructor
```

## Notes

- `Expr` is a one-field wrapper: match `(Expr.inner e)`, rebuild with `(Expr ...)`, copy spans.
- `ExprInner` has 83 variants; `Icnf` 19.

## See Also

- [icnf-new-form-needs-case](icnf-new-form-needs-case.md)
- [pass-copy-spans](pass-copy-spans.md)
