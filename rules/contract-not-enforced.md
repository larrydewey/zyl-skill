# contract-not-enforced

> Use `requires`/`ensures`/`invariant` for checked contracts (`E_CONTRACT_VIOLATION`, `result` in `ensures`); don't expect profiles, `checkpoint` rollback or typed `recover` arms.

## Why It Matters

Contracts are enforced (the file name is historical). They are lowered in `stdlib/compiler/expr_inner.zyl` (`contract-defn-body`, `contract-check`) to `(assert-true C "E_CONTRACT_VIOLATION: ...")`; the old unwired `contract_injection.zyl` was deleted. A false condition panics, and the panic unwinds to the nearest `try` like `error`. What each form does:

| Form | Implemented as |
|---|---|
| `(requires C)` | checked where written; false → `PANIC: E_CONTRACT_VIOLATION: precondition of f failed: (> b 0)` |
| `(invariant C)` | checked where written; message `invariant of f failed: C` |
| `(ensures C)` in a `defn` | the leading forms of the body (directly or in the body's `begin`); run after the body with its value bound to `result`; message `postcondition of f failed: C` |
| `(recover BODY ((Type) fallback) ...)` | `(try BODY (catch _ fallback))` with the **first** arm's fallback; the error type is not tested |
| `(checkpoint E)` | `E`; nothing saved or rolled back |
| `(contracts off FORM)` | `FORM` with every `requires`/`ensures`/`invariant` inside it removed (lexical: callees keep theirs) |
| bare top-level `(contracts off)` | strips the contracts of the next top-level form |
| any other `contracts` shape | compiles to nothing |
| profiles `strict/debug/warn/production` | none |
| top-level `invariant` | no effect |

A contract not directly in a `defn` body reports without the "of f" part (`precondition failed: C`).

## Bad

```lisp
(defn safe-div (a b)
  (/ a b))                        ; SIGFPE on b = 0, no message

(defn add1 (x)
  (begin
    (ensures (> result (count-calls)))   ; side effect runs on every call
    (+ x 1)))

(recover (parse s) ((ParseError) 0) ((IoError) -1))   ; always 0: arms are not matched by type
```

## Good

```lisp
(defn safe-div (a b)
  (begin
    (requires (> b 0))
    (ensures (<= result a))
    (/ a b)))

(safe-div 6 0)                          ; PANIC: E_CONTRACT_VIOLATION: precondition of safe-div failed: (> b 0)
(try (safe-div 6 0) (catch e -1))       ; -1
(contracts off (safe-div 6 3))          ; checks inside this form removed; safe-div's own remain
```

## Notes

- Conditions are ordinary code: typed, evaluated on every call, subject to every static check. Keep them pure and cheap.
- A tail call in a body that has `ensures` is no longer a tail call (the check runs after it), so deep recursion there uses stack.
- Contracts never alter type inference, ownership, regions or concurrency (spec P8).
- For recoverable failures a caller should handle, return `Result` instead ([err-result-for-expected-failures](err-result-for-expected-failures.md)).

## See Also

- [fn-unlowered-forms](fn-unlowered-forms.md)
- [err-no-assert-unwrap](err-no-assert-unwrap.md)
