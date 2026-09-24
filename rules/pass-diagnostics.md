# pass-diagnostics

> Report new errors through `err-at` (or `err-at-labels`) with a node and a catalogued code from `error_codes.zyl`, and warnings through `err-warn-at`, so they print located with a help line and work in JSON mode.

## Why It Matters

`error_codes.zyl` is the single-source catalog (name, phase, severity 1 error / 2 warning, default message). `error_report.zyl` renders every diagnostic:

| Helper | Use |
|---|---|
| `(err-at code msg fid off help)` | `error[CODE]: msg`, `--> file:line:col`, a 120-byte window of the source line, a caret, `= help:` |
| `(err-at-labels code msg fid off labels help)` | same, plus secondary spans; build a label with `(err-label-at node "text")` |
| `(err-warn-at code msg node help)` | a `warning[CODE]` through the runtime sink `zyl_warn_emit` (stderr, or the LSP's capture buffer); never `file-write 2` |
| `(err-suggest-help name candidates fallback)` | `did you mean `x`?` by edit distance, else `fallback` |

`fid`/`off` come from `(ffi-call "zyl_span_file" node 1000)` / `zyl_span_off`; a negative offset renders without a location. With `--error-format=json` (`zyl_diag_json`) every helper emits one JSON object instead, and `zyl_panic` wraps a bare `E_CODE: text` message the same way.

Located today: `E_MALFORMED_PARAMETER`, balance errors, `E_ARITY_MISMATCH`, `E_NON_EXHAUSTIVE_MATCH`, `E_UNREACHABLE_MATCH_ARM`, `E_DUPLICATE_DEFINITION`, `E_UNBOUND_VARIABLE` (with did-you-mean), `E_MUT_CONFLICT` and `E_CAPABILITY_LEAK` (labelled with the binding), `E_PKG_CAPABILITY_VIOLATION` (labelled with the definition), `E_INVALID_CHAR`, `E_TRAIT_NOT_FOUND`, `E_TRAIT_NOT_DERIVABLE`, `E_TYPE_MISMATCH`, the secret-checker errors and both `E_IMPL_FORBIDDEN` forms (`sc-fail-at`), and the unused/shadowing warnings. `E_INVALID_CAPABILITY`, `E_UNTERMINATED_STRING` and most `expr_inner` errors still print a bare `PANIC:` line.

## Good

- Add the code to `error_codes.zyl` (and spec §28 if it is a language error).
- Thread the offending node to `err-at`; include a concrete `= help:` fix. Pass a second node as a label when the error is about a relationship (a `set!` and its binding, a construct and its definition).
- Colocate fix-it text with the check (as `sb-hint` is with `sb-check-string`) so CLI, LSP and REPL share wording.
- Add a `tests/compile-fail/` case and, ideally, a regression test asserting location and hint.

## Notes

- Still open: colorized output; the LSP does not consume the JSON form yet.
- Never build a snippet by hand: a source line can be hundreds of KB, and padding built one character at a time is quadratic. `space-run` and `zyl_span_snippet` are already safe.
- Every check stops at its first error.

## See Also

- [pass-copy-spans](pass-copy-spans.md)
- [reference: error codes](../references/error-codes.md)
