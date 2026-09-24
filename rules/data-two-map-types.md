# data-two-map-types

> Pick one map per program: `core/map` (string keys, `Option` results, persistent) or `collections/map` (Int keys/values, default value, arena-backed).

## Why It Matters

Both modules define a type named `Map` and functions named `map-get`, `map-has`, `map-remove` with **different signatures and semantics**. The stdlib is one flat, fully visible namespace, so loading both invites collisions and confusion.

| | `core/map` | `collections/map` |
|---|---|---|
| Representation | association list of `(ME k v)` | parallel arena arrays |
| Keys / values | String keys (compared with `str-eq`), any value type: `(Map String V)` | Int keys, Int values |
| Create | `(map-new)` | `(map-create arena cap)` |
| Insert | `(map-insert m k v)` | `(map-put m k v)` |
| Lookup | `(map-get m k)` returns `Option`; `(map-get-or m k d)` | `(map-get m k default)` |
| Other | `map-entries`, `map-size`, `map-is-empty`, `map-map-values f m`, `map-from-list` | `map-len`, `map-cap`, `map-find`, `map-free` |
| Iteration order | deterministic, newest first (compiler uses it internally) | insertion order |
| `print` | `{b: 2, a: 1}` (`Show`) | address |

## Good

```lisp
(use core/map)
(defn count-service (m (svc String))
  (map-insert m svc (+ 1 (map-get-or m svc 0))))
```

## Notes

- `collections/collections` offers a third option, `Assoc` (`assoc-put k v m`, `assoc-get k default m`; collection **last**), for any key/value types.

## See Also

- [data-collections-persistent](data-collections-persistent.md)
- [det-ordered-collections](det-ordered-collections.md)
