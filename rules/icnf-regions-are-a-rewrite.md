# icnf-regions-are-a-rewrite

> Region inference is one conservative ICNF rewrite (`IVariant` → `IStackVariant`); keep any extension conservative.

## Why It Matters

`ri-transform-fns` runs after optimization. A let-bound `IVariant` becomes `IStackVariant` only when `ri-name-safe-in` (a total match over every `Icnf` constructor) proves every use is a `match` scrutinee or `print` argument and no nested `fn` references it. Undershooting costs an allocation; overshooting is silent memory corruption. The spec's full lattice (R1–R8, `E_REGION_ESCAPE`) is not implemented; `Region` survives only as the byte-buffer type parameter.

```lisp
(defn ri-transform-let (name val body)
  (let val2 (ri-transform-expr val)
    (let body2 (ri-transform-expr body)
      (match val2
        (IVariant _ _ _
          (if (> (ri-name-safe-in body2 name) 0)
            (ILet name (ri-to-stack-variant val2) body2)
            (ILet name val2 body2)))
        (_ (ILet name val2 body2))))))
```

## See Also

- [own-stack-promotion](own-stack-promotion.md)
- [pass-conservative-failure-direction](pass-conservative-failure-direction.md)
