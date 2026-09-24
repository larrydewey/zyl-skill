# contract-not-enforced

> Treat contracts as documentation: `requires`/`ensures` conditions are evaluated and discarded, and `invariant`, result binding, profiles and rollback do not exist.

## Why It Matters

The contract-injection pass (`contract_injection.zyl`) is not wired into the compiler. No check is ever injected and `E_CONTRACT_VIOLATION` is never raised. Conditions **are executed** (side effects run every call) and are subject to every static check. Some specified forms do not even compile.

| Form | Implemented as |
|---|---|
| `(requires C)` | `C` evaluated, result discarded |
| `(ensures C)` | same; **no** return-value binding — `result` is `E_UNBOUND_VARIABLE`, `(result)` an undefined function |
| `(invariant C)` | not recognized: undefined function `invariant` |
| `(recover BODY arms...)` | `BODY`; arms discarded |
| `(checkpoint E)` | `E`; nothing saved or rolled back |
| `(contracts off FORM)` | `FORM`; any other `contracts` shape compiles to nothing |
| profiles `strict/debug/warn/off/production` | none |

## Bad

```lisp
(defn safe-div (a b)
  (requires (> b 0))       ; violated precondition not reported
  (/ a b))                 ; SIGFPE on b = 0

(ensures (>= result 0))    ; E_UNBOUND_VARIABLE
```

## Good

```lisp
(defn safe-div (a b)
  (if (<= b 0)
    (error "safe-div: b must be positive")
    (/ a b)))
```

## Notes

- If you write `requires`/`ensures` for documentation, keep conditions pure and cheap.
- Contracts never alter type inference, ownership, regions or concurrency (spec P8) — true today trivially.
- Use `assert-true` (panics on false) or explicit `if`+`error` for checks that must hold; `Result` + `match` instead of `recover`.

## See Also

- [fn-unlowered-forms](fn-unlowered-forms.md)
- [err-no-assert-unwrap](err-no-assert-unwrap.md)
