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

## Compiler Commits (no co-authors)

1. `match: error on mixed literal and constructor patterns`
2. `test: fix exit status for test runner`
3. `when/unless: fix to be proper short-circuiting special forms`
4. `recover: warn when catch-all arm shadows specific error code arms`
5. `tuple: add (tuple ...) syntax with type checking, ICNF, and codegen`
6. Seed updates for each

## Skill Docs Commits

1. `docs: update skill for mixed match patterns diagnostic`
2. `docs: update skill for test exit status fix`
3. `docs: update pitfalls for buf-append and REPL regions`
4. `docs: update pitfalls for documented behaviors`
5. `docs: update skill for when/unless fix`
6. `docs: update skill for recover shadow warning`

## Test Status

- All 378 regression tests pass
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