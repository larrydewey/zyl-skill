# pass-total-structural-match

> In tree-rewriting passes, list every constructor explicitly; use `_` only in read-only checks that care about a few forms.

## Why It Matters

A rewriter with a catch-all silently passes a newly added constructor through unchanged (or, worse, rewrites it to a default). Listing every constructor makes a new constructor fail exhaustiveness in every rewriter until handled. Read-only checks can use `_` for the rest.

## Good

```lisp
; opt-expr-node in optimization.zyl, cut to its first arms
(use compiler/icnf)
(use compiler/optimization)
(defn my-expr (e)
  (match e
    (IConst _ e)
    (IStr _ e)
    (ISymAddr _ e)
    (IFlt _ e)
    (ILoad _ e)
    (IBinop op l r (IBinop op (my-expr l) (my-expr r)))
    (ICall name args (ICall name (my-list args)))
    (ICallClosure name args (ICallClosure name (my-list args)))
    (IFfi name args (IFfi name (my-list args)))
    (IPrint a (IPrint (my-expr a)))
    (IIf c t eb (IIf (my-expr c) (my-expr t) (my-expr eb)))
    (IWhile c b (IWhile (my-expr c) (my-expr b)))
    (ISet name v (ISet name (my-expr v)))
    (ILet name val body (ILet name (my-expr val) (my-expr body)))
    (ISeq es (ISeq (my-list es)))
    (IVariant nm tag fields (IVariant nm tag (my-list fields)))
    (IStackVariant nm tag fields (IStackVariant nm tag (my-list fields)))
    (IMatch scr arms (IMatch (my-expr scr) arms))
    (IFn name params body kinds (IFn name params (my-expr body) kinds))
    (ITryCatch tbody cvar cbody (ITryCatch (my-expr tbody) cvar (my-expr cbody)))
    (IRegion k b a l body (IRegion k b a l (my-expr body)))))
(defn my-list (es)
  (match es
    (Nil Nil)
    (Cons h t (Cons (opt-keep (my-expr h) h) (my-list t)))))
```

One arm per `Icnf` constructor (all 21), with no `_`: adding a constructor makes this match non-exhaustive until it is handled.

## Notes

- `Expr` is a one-field wrapper: match `(Expr.inner e)`, rebuild with `(Expr ...)`, copy spans.
- `ExprInner` has 79 variants; `Icnf` 21; the native backend's `MI` 33.
- The sound type checker enforces exhaustiveness in the compiler's own source too (`E_NON_EXHAUSTIVE_MATCH`), so a rewriter that lists every constructor fails the boot the moment one is added.
- Some total matches exist on purpose and must stay total: `icnf-has-set`, `ri-name-safe-in`, `rg-expr-node`, `opt-expr-node`, `mi-uses`. Others end in a deliberate `_` that is safe only because of a guard elsewhere: `ml-expr`'s `(_ (ml-const ms 0))` relies on `ml-ok` having declined the function ([cg-native-backend-mir](cg-native-backend-mir.md)).

## See Also

- [icnf-new-form-needs-case](icnf-new-form-needs-case.md)
- [pass-copy-spans](pass-copy-spans.md)
