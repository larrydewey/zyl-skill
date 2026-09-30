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

## Compiler Commits (no co-authors)

1. `match: error on mixed literal and constructor patterns`
2. `test: fix exit status for test runner`
3. `when/unless: fix to be proper short-circuiting special forms`
4. `recover: warn when catch-all arm shadows specific error code arms`
5. `tuple: add (tuple ...) syntax with type checking, ICNF, and codegen`
6. `test: add (:expect-panic bool) to the test form`
7. `test: take a test's options as sibling forms, and reject unknown ones`
8. Seed updates for each

## Skill Docs Commits

1. `docs: update skill for mixed match patterns diagnostic`
2. `docs: update skill for test exit status fix`
3. `docs: update pitfalls for buf-append and REPL regions`
4. `docs: update pitfalls for documented behaviors`
5. `docs: update skill for when/unless fix`
6. `docs: update skill for recover shadow warning`

## Test Status

- All 380 regression tests pass (378 + 2 new compile-fail cases)
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

## Error-as-value (spec 21.8) — why it is blocked

Spec §21.8 says `(error msg)` returns `(Err msg)` and does not throw. Making
it so is not a mechanical change:

- `error` is defined in `stdlib/allocator/allocator.zyl`, the lowest module —
  it has no `use` at all, and `core/result` (which declares `Ok`/`Err`)
  *depends on* allocator. So `error` cannot name `Err` without a cycle.
- The corpus treats `error` as a panic in 27 places across 22 files
  (5 stdlib, 15 tests, plus book/website examples). Flipping the meaning
  breaks every one of them: 33 regression tests fail at the type checker
  with `cannot unify Int with (Result ? String)`, because `error` is
  almost always in a value position expecting `Int`.
- `assert-fail` desugars to `(error msg)`, so it must move to the panicking
  form too, or it would stop failing.

Options, in rough order of cost:

1. **Move `error` up a level** into `core/result` (or a new `core/error`
   that uses it). Cheapest, but `error` stops being available to anything
   that only has the allocator, and it is prelude, so ordering matters.
2. **Make the compiler lower `error` itself** to an `Err` construction,
   the way it lowers `assert-equal`. Still needs `Err`'s tag present,
   so a program that never loads `core/result` has no `Err` to build.
3. **Give it its own builtin value type** (`(error msg)` yields a
   `Failure`, with `to-result` to convert). Departs from §21.8's wording
   but has no dependency problem, and it is the only option that does not
   make `error`'s type depend on what the program happens to import.

Option 3 is the one worth considering seriously; 1 is the cheapest if the
prelude can be reordered. This needs a decision before code, since it
changes what programs are allowed to write.
