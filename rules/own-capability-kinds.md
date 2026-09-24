# own-capability-kinds

> Know the capability kinds and which ones source code can actually produce: `TCap` (let/params), `TMut` (let-mut/for), `TPin` (ffi-pin result), and the `Secret` annotation.

## Why It Matters

Capabilities are inferred, never written. The type system represents them as `TCap CapKind Type` with `CapKind` ∈ `TCCap TCMut TCAtomic TCBox TCPin TCByte TCAtomicByte TCSecret`. Several have no source construct, so designs that assume them (atomic references, boxes) must use other tools.

| Spec | Meaning | Produced by |
|---|---|---|
| `TCap<T>` | shared immutable, any number of refs | every binding by default |
| `TMut<T>` | exclusive mutable, exactly one ref | `let-mut`, `for` variables |
| `TAtomic<T>` | atomic shared mutation | nothing; use `atomic/atomic` ops on addresses or `bytebuf-atomic-*` |
| `TBox<T>` | heap ownership | nothing; recursive fields are already pointers |
| `TPin<T>` | FFI-pinned | `ffi-pin` result |
| `Secret` | key material | `(k Secret)` parameter annotation, tracked by `secret_check` (taint), not the unifier |

## Operation matrix (spec)

| | TCap | TMut | TAtomic | TBox | TPin |
|---|---|---|---|---|---|
| read | yes | yes | yes | yes | yes |
| `set!` | no | yes | no | no | no |
| send to actor | yes | no | yes | no | no |
| pass to FFI | pinnable only | no | no | no | via ffi-pin |

Enforced today: the `set!` row, the send row for `let-mut` names, and the FFI pinnability check (partially).

## Notes

- Coercion: a `TMut` may be read as `TCap`; nothing upgrades a `let` to mutable. `Secret` and `Cap` unify both ways so keys flow through generic helpers.
- The Send predicate `tc-is-send` exists but no pass calls it.

## See Also

- [own-let-mut-only-set](own-let-mut-only-set.md)
- [secret-annotate-params](secret-annotate-params.md)
