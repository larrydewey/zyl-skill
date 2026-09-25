# own-with-region

> Wrap bulk temporary work in `(with-region (arena ...) body)` or `(with-region (fixed ...) body)` to release it when `body` ends and cap its memory; return only values built outside the region.

## Why It Matters

`with-region` is the region extension registry (spec §9): allocations in `body` that region inference places in the scope go to a region of a closed, audited kind, released when `body` ends. Unlike an `allocator/allocator` arena it holds ordinary typed values (lists, variants, strings), and unlike the automatic frame region it has an explicit, deterministic byte limit.

```lisp
(with-region (arena :block B :align A :limit L) body)   ; grows in B-byte blocks up to L bytes
(with-region (fixed :size S :align A) body)              ; exactly S bytes
```

- `B`: a multiple of 4096, at most 64 MiB. `L`: byte limit, 0 (the default) means none. `A`: a power of two from 8 to 4096, default 8.
- Any other kind, or a bad number, is a compile-time `E_REGION_SPEC`.
- Running out is `E_REGION_EXHAUSTED`, catchable with `try`, and deterministic: it depends only on the request sequence (blocks are page-aligned, so padding is the same every run).
- A value allocated in the region that reaches `body`'s result, or anything longer-lived (a global, a `send`, a foreign call), is a compile-time `E_REGION_ESCAPE`.

## Bad

```lisp
(defn leak () (with-region (arena :block 4096) (build 10 (WN))))   ; E_REGION_ESCAPE: the list is the result
(with-region (pool :block 4096) 1)                                  ; E_REGION_SPEC: no such kind
(with-region (arena :block 1000) 1)                                 ; E_REGION_SPEC: not a multiple of 4096
```

## Good

```lisp
(deftype WL (WN) (WC Int WL))
(defn build (n acc) (if (= n 0) acc (build (- n 1) (WC n acc))))
(defn wsum (l) (match l (WN 0) (WC h t (+ h (wsum t)))))

(defn arena-sum (n)
  (with-region (arena :block 65536 :align 16 :limit 1048576)
    (wsum (build n (WN)))))                  ; only the Int leaves the region

(defn safe-sum (n)                           ; recover from a full region
  (try (with-region (fixed :size 4096) (wsum (build n (WN))))
    (catch e (if (ffi-call "zyl_err_is" e "E_REGION_EXHAUSTED" 1000) -1 -2))))   ; zyl_err_is gives a Bool
```

## Notes

- Scopes nest; each gets its own annotation level (`4 + k` in attr table 4) and lowers to `IRegion kind block align limit body` (kind 1 arena, 2 fixed).
- Compiled code only: the REPL interpreter (`zyl eval`, `zyl repl`) ignores regions and allocates in its own arenas, so limits and `E_REGION_EXHAUSTED` are not enforced there.
- `(ffi-call "zyl_region_live_bytes" 1000)` reports bytes held by live regions, useful in tests that check memory is released.

## See Also

- [own-heap-never-freed](own-heap-never-freed.md)
- [own-regions-status](own-regions-status.md)
- [icnf-regions-are-a-rewrite](icnf-regions-are-a-rewrite.md)
