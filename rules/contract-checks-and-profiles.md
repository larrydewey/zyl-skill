# contract-checks-and-profiles

> Use `requires`/`ensures`/`invariant` for checked contracts (`E_CONTRACT_VIOLATION`, `result` in `ensures`), pick a profile with `--contracts=P` or `(contracts P)`, recover by error code with `recover`, and roll back `let-mut` state with `checkpoint`.

## Why It Matters

Contracts are enforced. They are lowered in `stdlib/compiler/expr_inner.zyl` (`contract-defn-body`, `contract-check`, `contract-profile`) to `(assert-true C "E_CONTRACT_VIOLATION: ...")` under the active profile; there is no separate injection phase. A false condition panics, and the panic unwinds to the nearest `try` like `panic`. What each form does:

| Form | Implemented as |
|---|---|
| `(requires C)` | checked where written; false → `PANIC: E_CONTRACT_VIOLATION: precondition of f failed: (> b 0)` |
| `(invariant C)` | checked where written; message `invariant of f failed: C` |
| `(ensures C)` in a `defn` | the leading forms of the body (directly or in the body's `begin`); run after the body with its value bound to `result`; message `postcondition of f failed: C` |
| `(recover BODY arm...)` | runs `BODY`; if it raises, arms are tried **in order**: `((E_CODE) fb)` matches an error whose message starts with that code, `((String) fb)` / any type-named arm / `(_ fb)` matches anything; no match re-raises |
| `(checkpoint E)` | `E`; if it raises, every outer `let-mut` variable `E` `set!`s gets its old value back, then the error is re-raised. Byte-buffer writes are **not** undone |
| `(contracts P FORM)` | `FORM` compiled under profile `P` (lexical: callees keep theirs) |
| bare top-level `(contracts P)` | profile `P` for the next top-level form |
| any other `contracts` shape | compiles to nothing |
| top-level `invariant` | no effect |

Profiles (`--contracts=P` on the command line sets the build's default; a directive overrides it for one form):

| Profile | A failed check |
|---|---|
| `strict` (default), `debug` | panics with `E_CONTRACT_VIOLATION` |
| `warn` | prints `warning: E_CONTRACT_VIOLATION: ...` on stderr and continues |
| `off`, `production` | clauses compiled out; the condition is never evaluated |

A contract not directly in a `defn` body reports without the "of f" part (`precondition failed: C`).

## Bad

```lisp
(defn safe-div (a b) (div! a b))                      ; stops on b = 0 with no word on whose fault it is

(defn guarded (a b)
  (recover (safe-div a b)
    ((String) 0)                                     ; catches everything, so the next arm never runs
    ((E_DIVISION_BY_ZERO) -1)))

(contracts production)
(defn f (a b) (begin (requires (> b 0)) (div! a b)))  ; check compiled out: back to E_DIVISION_BY_ZERO

(defn main () (begin (print (guarded 6 0)) (print (f 6 0)) 0))
```

This does not compile: the catch-all `(String)` arm ahead of a coded arm is `W_RECOVER_SHADOW`, which, despite its `W_` name, stops the build. Without that arm, `f` would still fail on `b = 0`: under `production` its `requires` is compiled out, so the zero reaches `div!` and the program stops with `E_DIVISION_BY_ZERO` instead of the precondition's message.

## Good

```lisp
(defn safe-div (a b)
  (begin
    (requires (> b 0))
    (ensures (<= result a))
    (div! a b)))

(defn main ()
  (begin
    (print (try (safe-div 6 0) (catch _ -1)))       ; -1
    (print (recover (safe-div 6 0)
      ((E_CONTRACT_VIOLATION) -1)                   ; the precondition failed
      (_ -2)))                                      ; anything else
    (print (contracts off (safe-div 6 3)))          ; 2: checks in this form removed, safe-div's own remain
    (let-mut x 10
      (begin
        (try (checkpoint (begin (set! x 20) (panic "fail"))) (catch _ 0))
        (print x)))                                 ; 10: rolled back
    (print (safe-div 6 0))                          ; PANIC: E_CONTRACT_VIOLATION: precondition of safe-div failed: (> b 0)
    0))
```

Output: `-1`, `-1`, `2`, `10`, then `PANIC: E_CONTRACT_VIOLATION: precondition of safe-div failed: (> b 0)` followed by the backtrace lines `  in safe-div` and `  in main`, exit status 1.

## Notes

- Conditions are ordinary code: each must be `Bool` (`(requires 1)` is `E_TYPE_MISMATCH`), evaluated on every call, subject to every static check. Keep them pure and cheap.
- Under `off`/`production` a clause is removed before type checking, so a stripped clause is not checked at all: `(requires (undefined-thing 1))` compiles. Build with the default profile at least once.
- `warn` output goes to stderr unbuffered, so it can appear before earlier stdout lines.
- `recover` catches panics (including `panic`) and contract violations, not `(Err ...)` values.
- `recover` arm order matters: a catch-all arm (`(String)` or `(_)`) before a specific error-code arm (`(E_...)`) is `W_RECOVER_SHADOW`, a located compile error despite the `W_` prefix.
- A `requires` does not make `/` total: the partial-operation check recognises no guard, so a division by a parameter is still `div!`/`div?` ([fn-division-total](fn-division-total.md)).
- A tail call in a body that has `ensures` is no longer a tail call (the check runs after it), so deep recursion there uses stack.
- Contracts never alter type inference, ownership, regions or concurrency (spec P8).
- For recoverable failures a caller should handle, return `Result` instead ([err-result-for-expected-failures](err-result-for-expected-failures.md)).

## See Also

- [fn-unlowered-forms](fn-unlowered-forms.md)
- [err-no-assert-unwrap](err-no-assert-unwrap.md)
