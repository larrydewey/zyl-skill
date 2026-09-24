# bits-only-8bit-widths

> Build 16/32/64-bit values from 8-bit loads and shifts (or `math/bits` packing helpers); the wider load/store names are reserved and rejected.

## Why It Matters

Only `load-u8`, `load-i8`, `store-u8`, `store-i8` are implemented. `load-u16`/`u32`/`u64`, signed variants and store counterparts are rejected with `E_RESERVED_KEYWORD` — the only place that code is raised today. The `:le`/`:be` selector is required even on byte widths so wider forms will read the same when they arrive.

## Bad

```lisp
(load-u32 :le buf 0)     ; E_RESERVED_KEYWORD
```

## Good

```lisp
(defn load-u32-le (buf off)
  (bit-or (load-u8 :le buf off)
          (shl (load-u8 :le buf (+ off 1)) 8)
          (shl (load-u8 :le buf (+ off 2)) 16)
          (shl (load-u8 :le buf (+ off 3)) 24)))
```

## See Also

- [bits-bytebuf-basics](bits-bytebuf-basics.md)
- [boot-bind-calls-before-binop](boot-bind-calls-before-binop.md) - in compiler code, pre-bind the loads
