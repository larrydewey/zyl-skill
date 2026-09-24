# pass-keep-kinds

> When a pass rebuilds an ICNF node, carry its codegen kind across with `ic-keep-kind` (and never read the type-annotation side tables of a node the type pass did not see).

## Why It Matters

`compiler/type_annotate.zyl` records each Expr node's type in runtime attr table 0 (`zyl_attr_set 0 node ty`). ICNF lowering (`ic-expr`) turns it into a kind on the new Icnf node in table 1 (`ic-mark-kind`); tables 2 and 3 hold trait-call renames and `print`-via-`Show` targets keyed by Expr node. The tables are keyed by node **address**: a rebuilt node starts with no entry, so its String/Float kind silently falls back to a word. `ic-hoist`, `opt-expr` and `ri-transform-expr` all wrap their results in `ic-keep-kind`.

## Good

```lisp
(defn my-pass (e)
  (ic-keep-kind (my-pass-node e) e))   ; keeps the original's kind unless the new node has one
```

## Notes

- Same shape as span copying ([pass-copy-spans](pass-copy-spans.md)); `ic-keep-span` only copies the span.
- The tables are cleared at the start of every `ta-annotate-program`, so a REPL entry never sees another entry's (freed) nodes; the parse-time field-type map (`zyl_smap_global 0`) is cleared at the start of `compile-to-exprs` and copies its keys.
- An Expr pass that runs *after* `ta-annotate` must not rebuild nodes, or it loses their types; add such passes before it.

## See Also

- [cg-kind-of](cg-kind-of.md)
- [pass-copy-spans](pass-copy-spans.md)
