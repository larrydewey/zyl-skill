# pass-diagnostics

> Report new errors through `err-at` (or `err-at-labels`) with a node and a code you add to `error_codes.zyl` and actually raise, warnings through `err-warn-at`, and type errors through `ta-type-error`, worded to `docs/diagnostics.md`, so they print located with a help line, reach the LSP, and work in JSON mode.

## Why It Matters

`error_codes.zyl` is the single-source catalog (name, phase, severity 1 error / 2 warning, default message), and it is exact: `verify/error-codes.sh` (the `scripts/error-codes` regression test) fails when a code is defined but raised nowhere, or raised but not defined. A raise is the code at the start of a string literal, after `PANIC: `, or inside `[...]`, outside comment lines. On 2026-10-02 the 31 codes nothing raised were removed, from the spec too, so a code that is planned but not raised does not belong in the catalog yet. `error_report.zyl` renders every diagnostic:

| Helper | Use |
|---|---|
| `(err-at code msg fid off help)` | `error[CODE]: msg`, `--> file:line:col`, a 120-byte window of the source line, a caret, `= help:` |
| `(err-at-labels code msg fid off labels help)` | same, plus secondary spans; build a label with `(err-label-at node "text")` |
| `(err-warn-at code msg node help)` | a `warning[CODE]` through the runtime sink `zyl_warn_emit` (stderr, or the LSP's capture buffer); never `file-write 2` |
| `(ta-type-error code msg node labels help)` | the type pass only: renders an `error[CODE]` with `err-diag`, sends it through `zyl_warn_emit`, counts it, and lets checking go on; `ta-fail-if-errors` then raises the closing line, `N errors; fix the first one first` (nothing after a single error), which the driver (`drv-report-failure`) prints plain, with no code and no `PANIC:`. `ZYL_STRICT_TYPES=report` turns each into a `W_TYPE_STRICT` warning and continues (for counting only) |
| `(err-suggest-help name candidates fallback)` | `did you mean `x`?` by edit distance (at most three, locals first via `err-suggest-help-scoped`; none for a one-letter name), else `fallback` |

`fid`/`off` come from `(ffi-call "zyl_span_file" node 1000)` / `zyl_span_off`; a negative offset renders without a location. With `--error-format=json` (`zyl_diag_json`) every helper emits one JSON object instead, and `zyl_panic` wraps a bare `E_CODE: text` message the same way.

Located today: almost every compile-time check, including the balance errors and `E_UNTERMINATED_STRING`, the reader's `E_INVALID_CHAR`/`E_INVALID_ESCAPE`/`E_INTEGER_OVERFLOW`, `E_MALFORMED_PARAMETER`, `E_MALFORMED_FORM`, `E_ARITY_MISMATCH`, the match checks (`E_NON_EXHAUSTIVE_MATCH`, `E_UNREACHABLE_MATCH_ARM`, `E_UNKNOWN_CONSTRUCTOR`, `E_NESTED_PATTERN`, `E_MATCH_ARM_COMPLEX`), `E_DUPLICATE_DEFINITION`, `E_MUT_CONFLICT` and `E_CAPABILITY_LEAK` (labelled with the binding), `E_INVALID_CAPABILITY`, `E_PKG_CAPABILITY_VIOLATION` (its help is the `(capabilities ...)` line to add), the secret-checker errors including `E_SECRET_UNANNOTATED`, `E_IMPL_FORBIDDEN`, the numeric check (`E_PARTIAL_OPERATION`), every type-pass error, and the unused/shadowing/`W_PANIC_UNMARKED` warnings. A located compile error prints without `PANIC:`; an unlocated one still prints `PANIC: CODE: ...`.

## The voice (`docs/diagnostics.md`)

Every message is held to a written standard, and `tests/scripts/diagnostics-voice.sh` pins the wording of a probe program, so a change to the voice is a change to that test:

- The head line is under 80 characters and names the user's own identifier and, where known, the value or type: `` `twice` takes an Int as its 1st argument (`n`), but this is a String ``, not `cannot unify Int with String`.
- At most one sentence of why, and only when the reason is security or not obvious (as a `= note:` line).
- The help is the exact line to write, as a lowercase imperative with no closing period: `` add `(capabilities ffi)` at the top of the file ``.
- Point at the offending expression, never the enclosing `defn` head; a bad `main` is reported at its last expression.
- No internal vocabulary: never "unify", "ICNF", "lower", "catch-all", pass names or node kinds. `zyl explain CODE` is the one pointer to more.
- One message per root cause: an unbound name is reported once, and nothing that depends on it is checked.
- Warnings in a standard-library module are hidden while compiling a program (`ZYL_WARN_ALL=1` shows them); errors never are.
- Output the program printed is flushed before the failure is reported.

A new code should also get a `zyl explain` entry (`stdlib/compiler/explain.zyl`) when it is one users will meet: a sentence, a minimal wrong program and its fix, which `tests/scripts/explain-examples.sh` compiles.

## Good

- Add the code to `error_codes.zyl` in the same change that raises it (and spec §28 if it is a language error); run `verify/error-codes.sh`.
- Thread the offending node to `err-at`; include a concrete `= help:` fix in the voice above. Pass a second node as a label when the error is about a relationship (a `set!` and its binding, a construct and its definition).
- Colocate fix-it text with the check (as `sb-hint` is with `sb-check-string`) so CLI, LSP and REPL share wording.
- Add a `tests/compile-fail/` case and, ideally, a regression test asserting location and hint.

## Notes

- The LSP turns on `zyl_warn_capture`, runs the checks and the type pass, and parses the captured text reports (`diagnostics-from-errors` in `stdlib/lsp/compiler_bridge.zyl`), so anything emitted through `zyl_warn_emit` with a location reaches the editor. A bare `zyl_panic` reaches it only as the one error that stopped the analysis.
- Still open: colorized output; the LSP reads the text form, not JSON. In JSON mode a panic message of the form `E_CODE: text` or `error[E_CODE]: text` becomes an object with that code (`zyl_panic_json`).
- Never build a snippet by hand: a source line can be hundreds of KB, and padding built one character at a time is quadratic. `space-run` and `zyl_span_snippet` are already safe.
- Every check except the type pass stops at its first error (a `zyl_panic`). The type pass reports all its errors, then fails; a new type rule should report through `ta-type-error` (or `ta-strict`, which picks the code from the message) and return a poisoned or fresh type so checking continues.

## See Also

- [pass-copy-spans](pass-copy-spans.md)
- [reference: error codes](../references/error-codes.md)
