# Zyl Session Summary - Completed Quick Wins

## Completed Fixes (7 pitfalls addressed)

| # | Issue | Fix | Status |
|---|-------|-----|--------|
| 4 | Mixed literal/constructor patterns in `match` | Added `E_MATCH_MIXED_PATTERNS` compile error | ✅ Done |
| 8 | `when`/`unless` evaluate body eagerly | Made them special forms with short-circuit evaluation | ✅ Done |
| 9 | `recover` arm order shadows specific errors | Added `W_RECOVER_SHADOW` warning | ✅ Done |
| 10 | `try` doesn't catch `Err` values | Documented as by-design (use `Result`/`Option`) | ✅ Documented |
| 24 | Test binary exit status 0 despite failures | Implicit `main` now returns `zyl_run_tests` result | ✅ Done |
| 29 | `buf-append` on non-fresh buffer accumulates | Already documented in `proj-buf-append-appends` | ✅ Documented |
| 30 | REPL ignores `with-region` limits | Documented as known limitation | ✅ Documented |
| 31 | Tuple syntax | Added `(tuple ...)` special form with type checking, ICNF lowering, and codegen | ✅ Done |
| 32 | No way to write a test that is *expected* to panic | `(:expect-panic true)` option on `test`, carried through `test-suite` | ✅ Done |
| 33 | `try`/`catch` unwinds rather than matching (spec 12.2) | Left as the one remaining deviation, but the type checker now enforces it: a `Result` body cannot be caught | ✅ Enforced |

## Compiler Commits (no co-authors)

1. `match: error on mixed literal and constructor patterns`
2. `test: fix exit status for test runner`
3. `when/unless: fix to be proper short-circuiting special forms`
4. `recover: warn when catch-all arm shadows specific error code arms`
5. `tuple: add (tuple ...) syntax with type checking, ICNF, and codegen`
6. `test: add (:expect-panic bool) to the test form`
7. `test: take a test's options as sibling forms, and reject unknown ones`
8. `error: return an Err, and add panic for the raising half`
9. Seed updates for each

## Skill Docs Commits

1. `docs: update skill for mixed match patterns diagnostic`
2. `docs: update skill for test exit status fix`
3. `docs: update pitfalls for buf-append and REPL regions`
4. `docs: update pitfalls for documented behaviors`
5. `docs: update skill for when/unless fix`
6. `docs: update skill for recover shadow warning`

## Test Status

- All 383 regression tests pass (378 + 2 option compile-fail + error-as-value + try-cannot-catch-err)
- Fixed point holds (verified with `./boot.sh`)

---

## Remaining Items (Major Features - Not Started)

These require significant architecture changes and are NOT quick wins:

| Priority | Item | Description |
|----------|------|-------------|
| Low | `let*` / named let | Loop-carried state |
| Low | Trait bounds / associated types / dyn | Major type system extension |
| Low | Error-as-value semantics | `error` returns `Err` instead of panicking |
| Low | Default public package index | Infrastructure |

## To Resume

```bash
cd /home/larry/git/larry/zyl
./run_regression_tests.sh --full
```

The repo is at a clean fixed point ready for the next session.

## Error-as-value (spec 21.8) — how it was done

`(error msg)` returns `(Err msg)` and raises nothing; the raising half is the
new `(panic msg)`, which keeps the old `error`'s body (`zyl_panic`) and its
intent.

The one real obstacle was the dependency cycle: `error` was in
`allocator/allocator.zyl`, the lowest module, and `core/result` — which
declares `Ok`/`Err` — depends on it. Chosen fix: `error` moved **up** into
`core/result.zyl`, beside the `Err` it constructs; `panic` stayed in the
allocator. `panic` joins the reserved names.

Consequence worth knowing: `error` is now in scope only where `Result` is, so
a program that explicitly loads `core/option` but not `core/result`/`core/core`
does not get it (26 stdlib/test modules are in that shape, though none call
`error`). Programs with no explicit core import get `core/core` implicitly and
are fine. Precedent: `map` also needs an import.

Cost: 22 files. 15 call sites in tests, 4 in stdlib (`result-expect`,
`option-expect`, `property-fail`, and `assert-fail`'s own rewrite, which has to
keep raising), and 3 in shell-embedded Zyl under `tests/scripts` — the last
were easy to miss, since they are not `.zyl` files.

`try`/`catch` is the one remaining deviation from spec §12.2: it still unwinds
instead of matching a `Result`. That is now *enforced* rather than merely
documented — body and handler must share one type, so
`(try (error "x") (catch e 7))` is `E_TYPE_MISMATCH` and `try` cannot be used to
intercept an `Err` at all. Pinned as `compile-fail/try-cannot-catch-err`.

Watch for: writing `error` where `panic` was meant. The type checker catches it
wherever a value of the other shape is expected, but **not** in statement
position, where the value is discarded — `(begin (error "x") (print "after"))`
compiles, prints, and exits 0.
