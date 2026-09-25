# data-two-map-types

> Pick one map per program: `core/map` (String keys, any values, `Option` results) or `collections/map` (Int keys and values, a default value, arena-backed).

## Why It Matters

Both modules define a type named `Map` and functions named `map-get`, `map-has`, `map-remove` with **different signatures and semantics**. Loading both is not an error: for each shared name, the module `use`d **later** silently wins, so a call written for the other one fails with a confusing `E_ARITY_MISMATCH` or `E_TYPE_MISMATCH` (or, worse, type-checks against the wrong one).

| | `core/map` | `collections/map` |
|---|---|---|
| Representation | ADT holding an association list of `(ME k v)` | struct of arena word arrays |
| Keys / values | String keys (compared with `str-eq`), any value type: `(Map String V)` | Int keys, Int values |
| Create | `(map-new)` | `(map-create arena cap)` (`arena`: an `Arena`), `(map-create-default cap)` |
| Insert | `(map-insert m k v)` | `(map-put m k v)` |
| Lookup | `(map-get m k)` returns `Option`; `(map-get-or m k d)` | `(map-get m k default)` |
| Other | `map-has`, `map-remove`, `map-entries`, `map-size`, `map-is-empty`, `map-map-values f m`, `map-from-list` | `map-has` (Bool), `map-remove`, `map-len`, `map-cap`, `map-find`, `map-free` |
| Iteration order | deterministic, newest first | insertion order |
| `print` | `{b: 2, a: 1}` (`Show`) | address (no `Show`) |

## Good

```lisp
(use core/map)
(defn count-service (m (svc String))
  (map-insert m svc (+ 1 (map-get-or m svc 0))))
(print (count-service (count-service (count-service (map-new) "a") "b") "a"))  ; {a: 2, b: 1}
```

## Notes

- `collections/collections` offers a third option, `Assoc` (`(assoc-empty)`, `assoc-put k v m`, `assoc-get k default m`; collection **last**), for any key/value types.

## See Also

- [data-collections-persistent](data-collections-persistent.md)
- [det-ordered-collections](det-ordered-collections.md)
