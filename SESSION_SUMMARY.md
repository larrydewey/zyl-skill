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
12. `len: one name for the length of a String, List, Vec, or Map`
13. `intmap: give the arena-backed Int map its own name, so Map means one thing`
14. `vec, map: the element and entry literals from spec 21.5`
15. `predicates: the six type predicates of spec 21.4`
16. `quote: a bracketed datum is the same datum as a parenthesized one`
17. Seed updates for each

## Skill Docs Commits

1. `docs: update skill for mixed match patterns diagnostic`
2. `docs: update skill for test exit status fix`
3. `docs: update pitfalls for buf-append and REPL regions`
4. `docs: update pitfalls for documented behaviors`
5. `docs: update skill for when/unless fix`
6. `docs: update skill for recover shadow warning`

## Test Status

- All 420 regression tests pass (378 + 28 new compile-fail + error-as-value + let* + tuples + len + vec/map literals + type predicates + quote brackets + int-literal range + operator arity + float range) (378 + 17 new compile-fail + error-as-value + let* + tuples + len + vec/map literals + type predicates + quote brackets)
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

## Working the spec's builtin list as a checklist

After the four items, I stopped adding features and instead took the spec's
§21 builtin list and *ran* every entry, which is how the next four gaps
were found. Probing beats reading: five of the entries I expected to be
fine were not, and two things I was sure were bugs were not.

What the probe found, beyond the four already done:

- `len` did not exist at all. The same idea had four names (`str-length`,
  `list-length`, `vec-len`, `map-size`) and nothing uniform. Now resolved
  during type checking to the argument type's own length function, so the
  four names still work.
- `int?`/`float?`/`bool?`/`string?`/`struct?`/`alias?` did not exist. They
  are decided in the type pass, which is the *only* option: a Zyl value is
  an untyped word, so there is no tag at runtime to test.
- `(vec ...)` and `(map ...)` did not exist.
- `'[1 2 3]` was rejected while `'(1 2 3)` was accepted — two spellings of
  one datum behaving differently.

What the probe cleared, which is worth as much:

- **`crc32c` was correct.** The spec says `Int Int -> Int` and the
  implementation agrees. I had reported a divergence from a bad test
  (`(crc32c 0 "a")` correctly fails — `x` is an Int, not a String).
- **`quote` was not missing.** Its `E_MALFORMED_FORM` on a name inside
  quoted data is exactly what the spec asks for; there is no symbol type.
  Only the bracket case was broken.
- **`mul-hi`/`mul-hi-u` are correct.** The bad result came from the literal
  `18446744073709551615`, which does not fit an Int64.

That last one is the useful lesson from this round. Probing found a real
gap and three false alarms, and the false alarms all came from the same
place: asserting a value without checking how it was produced. The literal
silently evaluating to 0 is a genuine defect, found by accident — see
below.

## Fixing what the probe found, and two more defects after it

**Integer literals: silent zero.** `9223372036854775808` evaluated to `0`,
and `-9223372036854775809` wrapped to a *positive* value of the wrong sign.
`zyl_cstr_to_int_base` answers 0 for a value it cannot hold, so a literal
too large simply became the number zero and the program carried on. Now
`E_INTEGER_OVERFLOW`, reported from the token stream before parsing, in
every base — the radix forms had the same hole. Four bugs of my own got
there, each looking plausible:

- `(- c 97 87)` parses as `(c-97)-87`, not `c-87`. `-` took three arguments
  with no arity error, so `'f'` scored -82 and the comparison ran backwards:
  `0xffff000000000000` passed and `0x7F00000000000000` was rejected. A
  check wrong in *both* directions looks like it is working.
- Testing `<= 102` before `<= 70` catches `'F'` in the `a-f` branch.
- `'a'` is ten, not zero.
- The underlying one: `str-length` on a substring of a substring of a *view*
  reports a length that is wrong either way. Work on indices, not
  substrings.

**Two `Map` types, colliding on three function names.** `core/map` is a
generic `Map<K, V>`; `collections/map` was an arena-backed Int table. They
also both exported `map-get`/`map-remove`/`map-has` with incompatible
signatures. Importing both compiled clean, and the shared names silently
resolved to the Int-only one — so the generic versions became unreachable
behind an arity error that mentioned neither `Map` nor the collision. The
type checker cannot catch this: both candidates are well-typed, so it is
the *name* that is wrong, not the program. The Int one is `IntMap` now.

The rename also exposed that `core/map`'s `map-has` returned `1`/`0` rather
than a `Bool`, which eight LSP call sites had grown a `(> (map-has ...) 0)`
workaround for. Renaming the type was not enough; the three shared function
names are what actually collided.

**`E_FLOAT_OVERFLOW` deleted, not implemented.** A float literal out of
range is `inf` or `0.0` — IEEE, and what C/Python/JS/Rust produce. The
opposite of the integer case, where the alternative was a silent 0 or a
silent wrap: `inf` announces itself. A catalogued code that nothing raises
implies a check that does not exist, so it is gone and the header says why.

## Three things I reported as bugs that were not

The session's clearest lesson, and all three were the same mistake: asserting
from the spec text without checking what the code does.

1. **`crc32c`** — reported as a signature divergence. The spec says
   `Int Int -> Int` and the code agrees; my test passed a String.
2. **`quote`** — reported as "not implemented". It is implemented, and its
   `E_MALFORMED_FORM` on a name inside quoted data is exactly what the spec
   asks for. Only the bracket case was broken.
3. **n-ary `-`** — reported as a bug because §21.1 writes `(- a b)`. The
   book has said "Arithmetic (n-ary, left-associative)" all along,
   `regression/arithmetic.zyl` pins `(- 10 2 2 2 2)` = 2 in a test named
   `int-modulo-like` (chained subtraction is how you spell modulo), and
   enforcing the spec would have broken four real call sites in codegen.zyl
   before the compiler even finished building.

Also **`mul-hi`/`mul-hi-u` are correct** — the bad result came from the
literal `18446744073709551615`, which does not fit an Int64. That is what
led to the integer-literal bug above.

So: absence from a spec list is not evidence. The §2 `Expr :=` grammar omits
`defstruct`, `trait`, `impl`, `set!` and plain `(let x e1 e2)`, all of which
exist. The §1.3 keyword list *was* exhaustive-looking and did omit `let*`.

## A process failure worth naming

Five commits claimed to add notes to `spec/02-syntax-and-forms.md` and did
not. Every one was a `str.replace` whose anchor was not in the file, and
Python's `str.replace` is not an error when the pattern is absent — it
returns the string unchanged. I verified those commits by test output and
never checked the file. The table rows landed (a different replace, whose
anchor matched); all seven prose notes did not. `b48162a` claimed "the
notation is now called out in spec/02" and that was false when made.

Now every insertion asserts its anchor appears exactly once and its own text
appears exactly once after.

## Two features deliberately not built

**Trait bounds (§6.1).** Would be *additive* — a colon distinguishes
`((T : Ord) a)` from `((x Int) a)`, so none of the 3,867 typed parameters in
166 files break — and the machinery is mostly there: a `Trait.method` call
on a type parameter already resolves per specialization. What is missing is
§6.4's case, a parameter with no call-site evidence, which needs a finite
candidate set from the program's impls. Deferred deliberately; the spec's
own "Implemented:" section already documents the current state and the
rejected forms have precise diagnostics.

**Recursive named `let`.** Blocked on a measured property: **closures
capture by value.** A closure over a `let-mut` local still reads the value
from before the `set!`. So binding the name before the initializer — the
obvious one-line fix — would compile and then capture an uninitialized
slot. The two real designs (by-reference capture, or a one-slot mutable
cell, which the language has no primitive for) are both language-wide.

## Two stdlib defects the probe uncovered

**Two types named `Map`.** `core/map` is a generic `Map<K, V>`; an
arena-backed Int-to-Int one lived in `collections/map`, and they also
shared `map-get`, `map-remove` and `map-has` with incompatible
signatures. Nothing stopped a program importing both: it compiled clean and
the shared names silently resolved to the Int-only one, so the generic
versions became unreachable behind an arity error. The losing side of a
name collision that is not diagnosed is the worst shape one can take. The
Int one is now `IntMap` in `collections/intmap`.

`duplicate_check.zyl` had claimed the two were "never `use`d together" and
used that to justify checking names only within one resolved program. That
was an assumption about which modules those were, and this pair was the
counterexample — nothing enforced it. The comment now says so.

The rename also exposed that `core/map`'s `map-has` returned `1`/`0`
instead of a Bool, which eight LSP call sites had grown a `(> (map-has
...) 0)` workaround for.

**Out-of-range integer literals evaluate to 0.** `9223372036854775808`
prints `0`; `-9223372036854775809` prints `9223372036854775807`. This one
is **not fixed** — it is a silent wrong answer, which is the worst class of
bug, and it is a lexer fix (a range check on the literal text, plus a token
for the overflow so the diagnostic has a position) that deserves its own
commit and its own review. It is the first thing I would do next.

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
