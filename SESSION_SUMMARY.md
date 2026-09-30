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
| 31 | Tuple syntax | `(tuple ...)` + `(tuple-get t i)`, element types part of the type (spec 21.5) | ✅ Done |
| 32 | No way to write a test that is *expected* to panic | `(:expect-panic true)` option on `test`, carried through `test-suite` | ✅ Done |
| 33 | `let*` unavailable | Sugar for nested `let`, rewritten in `desugar.zyl` | ✅ Done |
| 34 | `try`/`catch` unwinds rather than matching (spec 12.2) | Left as the one remaining deviation, but the type checker now enforces it: a `Result` body cannot be caught | ✅ Enforced |

## Compiler Commits (no co-authors)

1. `match: error on mixed literal and constructor patterns`
2. `test: fix exit status for test runner`
3. `when/unless: fix to be proper short-circuiting special forms`
4. `recover: warn when catch-all arm shadows specific error code arms`
6. `test: add (:expect-panic bool) to the test form`
7. `test: take a test's options as sibling forms, and reject unknown ones`
8. `error: return an Err, and add panic for the raising half`
9. `let*: sequential bindings, sugar for nested let`
10. `Revert the tuple stub: it built values nothing could read`
11. `tuple: build and read tuples, with the element types in the type`
12. Seed updates for each

## Skill Docs Commits

1. `docs: update skill for mixed match patterns diagnostic`
2. `docs: update skill for test exit status fix`
3. `docs: update pitfalls for buf-append and REPL regions`
4. `docs: update pitfalls for documented behaviors`
5. `docs: update skill for when/unless fix`
6. `docs: update skill for recover shadow warning`

## Test Status

- All 391 regression tests pass (378 + 6 new compile-fail + error-as-value + let* + tuples)
- Fixed point holds (verified with `./boot.sh`)

---

## Remaining Items (Major Features - Not Started)

These require significant architecture changes and are NOT quick wins:

| Priority | Item | Description |
|----------|------|-------------|
| Low | named let | Blocked: needs a closure that rebinds captured state, which `E_MUT_CONFLICT` forbids. `let*` is done |
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

## Tuples: how they ended up working

The first attempt was wrong and was reverted: `(tuple ...)` lowered to an
`IVariant` with a made-up tag equal to the arity, and a tuple value could not
be read at all — no `match` arm could name it, because the constructor was
never registered in the variant table icnf matches against. Removing it left
the tree matching what the book already said, but "absent" was the wrong
answer too.

The working version (614e8ab) treats a tuple as what it is — a generated
single-variant ADT, one per element type list, named `__Tuple<arity>~<types>`.
So the element types are part of the type, `(tuple 1 2)` and
`(tuple "a" "b")` cannot be unified, and a positional read is type-checked
rather than a raw word.

This could not be a parse-time desugar, which is the thing the first attempt
got wrong: the element types are unknown until the type pass. So the name is
derived in `type_annotate.zyl`, the `deftype` that gives the variant a tag is
pushed onto `ta-extra` (which `ta-annotate` returns as part of the program, so
`ic-collect-vt` registers it), and it is marked as a struct so the tag is
globally unique — each generated type numbers its own tags from 0, and a match
on one must not fire on another.

`tuple-get` reuses `struct-get`'s `ta-pending` deferral. Tuple reads are tagged
`"@t"` in the pending because `@` cannot appear in an identifier: the struct
path routes an unresolved receiver through a "sole struct with this field"
heuristic that knows nothing about tuples, and without the tag a generic
receiver got that heuristic's wrong message.

**Known limit:** a generic function cannot take a tuple. Nothing can say a type
parameter is a tuple (no trait bounds, spec 6.1), so `tuple-get` on one is
`E_CANNOT_INFER`. Concrete tuples, nesting, returning them, and mixed element
types all work.

## Verified conformant, no change needed

Probed rather than assumed:

- `recover` (§23.4): all four arm rules — `E_` code matched by message prefix,
  `_` and type-named arms matching anything, arms tried in order, and an
  unmatched error propagating.
- `checkpoint` (§23.5): restores every outer `let-mut` the body assigned, before
  the error propagates.
- Contract profiles (§23): `warn` reports and continues, `off`/`production`
  compile the clause out, `(contracts P form)` and a bare `(contracts P)` both
  override for the following form. `--contracts=warn` and
  `(contracts warn (defn ...))` both work.

  One trap worth recording: `(contracts warn (print (f 0)))` does **not** relax
  `f`'s already-compiled `requires`. The override applies to the form it wraps,
  so it has to wrap the *definition*. A call site cannot retroactively change how
  a body was lowered.
