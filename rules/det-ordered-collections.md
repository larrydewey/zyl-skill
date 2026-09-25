# det-ordered-collections

> Iterate only ordered structures (lists, association lists, insertion-ordered arrays); a hash table may be probed by key but never iterated.

## Why It Matters

Output must be a pure function of input. Iterating a hash table exposes hash seeds or allocation order. The compiler's tables are lists built in source order or sorted by name (`qualify.zyl`); `core/map` is an association list; `collections/map`/`set` scan insertion-ordered arrays. The runtime's source-span table and the interpreter's FNV-1a function table are hash tables that are only **probed by key**.

## Good

```lisp
(use core/map)
(map-entries m)           ; deterministic order
```

## Bad (in compiler or program logic)

```text
iterate a table keyed by node address or pointer value
name generated symbols from heap addresses
```

## Notes

- `stdlib/core/map.zyl` (String keys, compared with `str-eq`) iterates most-recently-inserted first; re-inserting a key moves it to the front.
- Historical fixed-point breakers: generated names from heap pointers, string `=` comparing pointers, iteration of address-keyed tables.

## See Also

- [det-no-address-dependent-output](det-no-address-dependent-output.md)
- [data-two-map-types](data-two-map-types.md)
