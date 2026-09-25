# own-capability-kinds

> Know that capabilities are not part of the type checker: `TCap`/`TMut` are decided by binding form (`let` versus `let-mut`/`for`) in `mutability_check`, `Secret` by taint in `secret_check`, and the only capability-like type is `(Pin a)`, the result of `ffi-pin`.

## Why It Matters

The spec describes capability types (`TCap<T>`, `TMut<T>`, `TAtomic<T>`, `TBox<T>`, `TPin<T>`). The implementation splits them up. The sound Hindley-Milner checker (`type_annotate.zyl`) works on plain types (Int, Float, Bool, String, Unit, ADTs, functions and handle types) and knows nothing of TCap or TMut; the old `CapKind` machinery went with `type_inference.zyl`, which was deleted. What is enforced lives in separate syntactic passes, and some spec kinds have no source construct at all, so designs that assume them (atomic references, boxes) must use other tools.

| Spec | Meaning | In the implementation |
|---|---|---|
| `TCap<T>` | shared immutable, any number of refs | every `let` binding and parameter (`mutability_check`) |
| `TMut<T>` | exclusive mutable, exactly one ref | `let-mut` and `for` variables; `set!` on anything else is `E_MUT_CONFLICT` |
| `TAtomic<T>` | atomic shared mutation | no type; `atomic/atomic` operations on a `Ptr`, or `bytebuf-atomic-*` |
| `TBox<T>` | heap ownership | nothing; recursive fields are already pointers |
| `TPin<T>` | FFI-pinned | the type `(Pin a)` of `(ffi-pin v)`; `ffi-unpin` takes it back to `a` |
| `Secret` | key material | `(k Secret)` parameter or field annotation, `(Secret T)`, or `(impl Secret T ...)`; tracked by `secret_check` (taint). To the type checker a `Secret` annotation is transparent |

## Operation matrix (spec) and what is enforced

| | TCap | TMut | TAtomic | TPin |
|---|---|---|---|---|
| read | yes | yes | yes | via `ffi-unpin` |
| `set!` | no (`E_MUT_CONFLICT`) | yes | no | no |
| send to actor | yes | no (`E_CAPABILITY_LEAK`) | yes | not checked |
| pass to FFI | as its extern type | as its extern type | no | as `(Pin a)` |

Enforced: the `set!` row; the send row for `let-mut` names in `send` and `spawn` (by name, not by type); `Secret` crossing an actor (`E_SECRET_ESCAPE`) or FFI unpinned (`E_FFI_PIN_REQUIRED`); a function passed to `ffi-pin` (`E_FFI_TYPE_NOT_PINNABLE`) or a closure passed to a foreign call (`E_INVALID_CAPABILITY`).

## Notes

- Nothing upgrades a `let` to mutable, and capabilities are never written in source: `(x TMut)` is not an annotation.
- There is no Send predicate on types: a closure or a `(Pin a)` in a message passes when it is bound by plain `let`.
- Package capabilities (`(capabilities io ffi actor ...)` in `zyl.pkg`) are a different thing; see [pkg-capabilities](pkg-capabilities.md).

## See Also

- [own-let-mut-only-set](own-let-mut-only-set.md)
- [actor-no-let-mut-crossing](actor-no-let-mut-crossing.md)
- [secret-annotate-params](secret-annotate-params.md)
