# data-two-map-types

> Choose the map by key type: `core/map` (String keys, any values, `Option` results) or `collections/intmap` (Int keys and values, a default value, region-backed); their names no longer overlap, so both may be loaded together.

## Why It Matters

The two modules used to define a type named `Map` and functions named `map-get`, `map-has` and `map-remove` with different signatures, so loading both silently let the later one win. The Int-keyed one is now `IntMap` with `intmap-*` functions, and `Map`, `len` and the `(map key val ...)` literal all mean `core/map`. Picking the wrong one is still a type error rather than a choice: `core/map` keys are compared with `str-eq`, so an Int key is `E_TYPE_MISMATCH`.

| | `core/map` | `collections/intmap` |
|---|---|---|
| Representation | ADT holding an association list of `(ME k v)` | struct of word arrays in the caller's region |
| Keys / values | String keys (compared with `str-eq`), any value type: `(Map String V)` | Int keys, Int values |
| Create | `(map-new)` | `(intmap-new cap)` (`cap` a hint) |
| Insert | `(map-insert m k v)` | `(intmap-put m k v)` |
| Lookup | `(map-get m k)` returns `Option`; `(map-get-or m k d)` | `(intmap-get m k default)` |
| Other | `map-has`, `map-remove`, `map-entries`, `map-size`, `map-is-empty`, `map-map-values f m`, `map-from-list` | `intmap-has` (Bool), `intmap-remove`, `intmap-len`, `intmap-cap`, `intmap-find`, `intmap-free` |
| Iteration order | deterministic, newest first | insertion order |
| `print` | `{b: 2, a: 1}` (`Show`) | address (no `Show`) |

## Good

```lisp
(use core/map)
(use collections/intmap)

(defn count-service (m (svc String))
  (map-insert m svc (+ 1 (map-get-or m svc 0))))

(defn main ()
  (let ids (intmap-put (intmap-put (intmap-new 4) 3 30) 4 40)
    (begin
      (print (count-service (count-service (count-service (map-new) "a") "b") "a"))  ; {a: 2, b: 1}
      (print (intmap-get ids 3 -1))                                                   ; 30
      (print (intmap-get ids 9 -1))                                                   ; -1
      0)))
```

## Notes

- `collections/collections` offers a third option, `Assoc` (`(assoc-empty)`, `assoc-put k v m`, `assoc-get k default m`; collection **last**), for any key/value types.

## See Also

- [data-collections-persistent](data-collections-persistent.md)
- [det-ordered-collections](det-ordered-collections.md)
