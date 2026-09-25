---
name: zyl
description: >
  Expert Zyl knowledge for writing, reviewing and debugging Zyl code and the
  self-hosted Zyl compiler. 180 rules in 26 categories across five tiers
  (foundations, language model, systems, engineering, compiler internals),
  prioritized by impact, plus reference tables for error codes, built-ins,
  the standard library, the pipeline and spec-vs-implementation status.
  Use when editing any *.zyl file or zyl.pkg manifest, answering questions
  about Zyl syntax or semantics, reviewing Zyl code, working in
  stdlib/compiler or selfhost/, or diagnosing boot/fixed-point failures.
triggers:
  - .zyl files, zyl.pkg, zyl.lock
  - zyl, zyl-self, zyl-lsp, zyl repl, zyl eval, zyl doc
  - selfhost, stage1, stage2, stage3, boot.sh, fixed point, reseed
  - stdlib/compiler, icnf, codegen, ic-, cg-, mr-, sb-
metadata:
  version: "3.0.0"
  sources:
    - book/src (The Zyl Programming Language, Parts I-V and Appendices A-F)
    - zyl_specification.txt v5.0
    - stdlib/compiler, selfhost, runtime (implementation authority)
---

# Zyl Expert Guide

Zyl is a deterministic Lisp systems language: S-expression syntax, sound Hindley–Milner type checking, capability rules (TCap/TMut), region-based memory with in-place reuse, actors, a custom IR (ICNF), and x86_64 native code from a register-allocating backend. The compiler is written in Zyl and reproduces itself byte for byte (`./boot.sh`). Strict left-to-right evaluation; same input, same output.

**The spec is the design; the compiler is the truth.** The type checker is sound, so most mistakes are compile errors now, but parts of the spec are still unimplemented and a few gaps fail *silently* (they compile fine and compute the wrong result; see [pitfalls](references/pitfalls.md)). Every rule here states current behavior. When a rule and the spec disagree, follow the rule; when a rule and the code disagree, the code wins — then fix the rule.

## How This Skill Is Organized

- **`SKILL.md`** (this file): the ten facts to internalize, the category table, and a one-line index of every rule.
- **`rules/<prefix>-<slug>.md`**: one rule per file — *Why It Matters*, *Bad*, *Good*, notes, cross-links. Read the file before relying on a rule.
- **`references/`**: dense lookup tables —
  [pitfalls](references/pitfalls.md) (silent-failure checklist; read before any review) ·
  [error-codes](references/error-codes.md) ·
  [builtins](references/builtins.md) ·
  [stdlib](references/stdlib.md) ·
  [implementation-status](references/implementation-status.md) ·
  [pipeline](references/pipeline.md) ·
  [debugging](references/debugging.md) ·
  [migration](references/migration.md) (from Rust/C/Lisp).

Tiers build on each other: Tier 1 applies to every line of Zyl, Tier 5 only when changing the compiler itself.

## The Ten Facts

1. **Type checking is sound and strict.** Every type error in the program is reported (`E_TYPE_MISMATCH`, `E_INFINITE_TYPE`, `E_CANNOT_INFER`, `E_UNBOUND_VARIABLE`), then the compile fails. Conditions are `Bool`, arithmetic is `Int` or `Float` with no mixing and no implicit conversion, statements are `Unit` (the literal is `unit`), both branches of an `if` share a type, and `main` is `() -> Int`. Every trait call resolves statically. The known holes: `receive`'s result is not checked, and a lowercase field type in `deftype` is not a type parameter. → [type-sound-checking](rules/type-sound-checking.md)
2. **Lists have literal syntax; most other punctuation is invalid.** `[a b c]`, `(list a b c)` and `'(1 2 3)` build a `List` of one element type, and `` `(1 ,x ,@xs) `` fills in holes. A name inside quoted data, and a `,`/`,@` outside a quasiquote or macro template, is `E_MALFORMED_FORM`. `@ # $ | ^ \`, a lone `.` and non-ASCII bytes outside strings and comments are a located `E_INVALID_CHAR`. → [syn-list-literals-and-quote](rules/syn-list-literals-and-quote.md), [syn-no-stray-characters](rules/syn-no-stray-characters.md)
3. **Matches: spell constructors exactly, one level at a time, `_` last.** A nested pattern is `E_NESTED_PATTERN` and a non-exhaustive match is an error, but an arm head that is not a known constructor is still a silent catch-all: a misspelled constructor with no fields, or one of your own such constructors in a field slot, binds a variable. → [match-misspelled-last-arm](rules/match-misspelled-last-arm.md)
4. **Some things still compile and do the wrong thing.** `exit` returns 0 and the program carries on, `read-line` gives a null String, `close` and `with-resource` cleanup do nothing, and `alias` makes a type variable. `print` of a record with no `Show` prints its address, and `(print true)` prints `1`. Integer arithmetic wraps, and division by zero kills the process with SIGFPE. → [fn-unlowered-forms](rules/fn-unlowered-forms.md), [references/pitfalls](references/pitfalls.md)
5. **Mutation is only `set!` on a `let-mut` name.** Params and `let` are immutable, struct fields are immutable (rebind the whole value), closures capture by value and cannot `set!` captures. → [own-let-mut-only-set](rules/own-let-mut-only-set.md)
6. **Only `if` branches and `match` arms need `begin` for several forms.** `defn`, `let`, `fn`, `while`, `for`, `cond` clauses and `catch` handlers sequence their forms. An `if` silently drops forms after its else branch, and `test` and `defmacro` take exactly one body form. → [fn-begin-multi-form-bodies](rules/fn-begin-multi-form-bodies.md)
7. **Every foreign symbol needs an `extern`, and every `ffi-call` a timeout.** Declare `(extern "sym" (ParamType ...) ResultType)` with word-sized types (a `Float` crosses only as its bits). End each `ffi-call` with a positive integer literal timeout in milliseconds: it is checked at compile time and enforced at run time (`E_FFI_TIMEOUT`, and the C function is abandoned). `ffi-pin` gives a `(Pin a)`, a pointer to a slot. → [ffi-extern-required](rules/ffi-extern-required.md), [ffi-timeout-always-last](rules/ffi-timeout-always-last.md)
8. **Actors exchange data with `send` + `(receive)`.** `(actor-self)` gives an id to reply to (main included). A spawn entry takes no parameters, and a `let-mut` capture is `E_CAPABILITY_LEAK`. Closure messages cannot be sent from a Zyl program. An actor that never calls `receive` drops its messages, and returning from `main` drains every actor. → [actor-send-is-discarded](rules/actor-send-is-discarded.md)
9. **Memory is regions, not a collector.** Per-call frame regions reclaim what the compiler proves short-lived, and the reuse pass updates a unique, dead value in place. Values that escape to the heap live until exit. Use `with-region` for bulk temporaries, and views (`text/view`, `collections/slice`) instead of copying substrings and sub-vectors. → [own-heap-never-freed](rules/own-heap-never-freed.md), [data-views-and-slices](rules/data-views-and-slices.md)
10. **Compiler changes must reach a new fixed point:** `./boot.sh --bootstrap-from-self && ./boot.sh`, then commit the seed (a verified `./boot.sh` also refreshes `~/.zyl`). The native backend (ICNF, then MIR, then linear-scan registers) compiles most functions, and the stack machine compiles the rest. `ZYL_MIR=0`, `ZYL_INLINE=0`, `ZYL_REUSE=0` and `ZYL_REGIONS=0` bisect a miscompile. Introduce new syntax, or a new runtime function the compiler calls, in two steps. → [boot-fixed-point-workflow](rules/boot-fixed-point-workflow.md), [cg-native-backend-mir](rules/cg-native-backend-mir.md)

## Minimal Correct Program

```lisp
;; hello.zyl  --  zyl hello.zyl -o hello && ./hello
(deftype Shape (Circle Int) (Rect Int Int))

(defn area (s)
  (match s
    (Circle r (* 3 (* r r)))
    (Rect w h (* w h))))

(defn label (name n)                         ; types inferred: name is a String
  (begin
    (print (str-concat name ":"))
    (print n)))

(defn main ()
  (begin
    (label "circle" (area (Circle 2)))
    (label "rect" (area (Rect 3 4)))
    0))                                      ; main's value is the exit status
```

Tests live in a separate file: top-level `(test "name" (assert-equal actual expected))` forms and a final `(run-tests)`, no `main`.

## Rule Categories by Priority

Impact: **CRITICAL** = silent miscompile, wrong result or crash; **HIGH** = error you will hit constantly or a core idiom; **MEDIUM** = important knowledge; rules tagged **[CRITICAL]** below are the individually most dangerous ones.

| Tier | # | Category | Impact | Prefix | Rules |
|---|---|---|---|---|---|
| 1 | 1 | Syntax & Lexical Structure | CRITICAL | `syn-` | 7 |
| 1 | 2 | Functions, Bindings & Control Flow | CRITICAL | `fn-` | 15 |
| 1 | 3 | Structs, ADTs & Collections | CRITICAL | `data-` | 12 |
| 1 | 4 | Pattern Matching | CRITICAL | `match-` | 8 |
| 1 | 5 | Error Handling | CRITICAL | `err-` | 5 |
| 1 | 6 | Contracts | HIGH | `contract-` | 1 |
| 2 | 7 | Ownership, Capabilities & Regions | HIGH | `own-` | 8 |
| 2 | 8 | Type System | CRITICAL | `type-` | 3 |
| 2 | 9 | Generics | HIGH | `gen-` | 6 |
| 2 | 10 | Traits | HIGH | `trait-` | 5 |
| 2 | 11 | Closures | HIGH | `closure-` | 3 |
| 2 | 12 | Macros | HIGH | `macro-` | 9 |
| 3 | 13 | FFI | CRITICAL | `ffi-` | 7 |
| 3 | 14 | Actors | CRITICAL | `actor-` | 7 |
| 3 | 15 | Bits, Bytes & Buffers | HIGH | `bits-` | 7 |
| 3 | 16 | Secrets & Constant-Time | CRITICAL | `secret-` | 7 |
| 3 | 17 | Cryptography Library | MEDIUM | `crypto-` | 2 |
| 4 | 18 | Testing | HIGH | `test-` | 7 |
| 4 | 19 | Modules & Packages | HIGH | `pkg-` | 11 |
| 4 | 20 | Determinism | HIGH | `det-` | 4 |
| 4 | 21 | Tooling (CLI, REPL, LSP) | MEDIUM | `tool-` | 5 |
| 4 | 22 | Project Idioms | MEDIUM | `proj-` | 3 |
| 5 | 23 | Bootstrap & Fixed Point | CRITICAL | `boot-` | 10 |
| 5 | 24 | ICNF | MEDIUM | `icnf-` | 6 |
| 5 | 25 | x86_64 Codegen | HIGH | `cg-` | 9 |
| 5 | 26 | Writing Compiler Passes | HIGH | `pass-` | 13 |
## Tier 1 — Foundations (every program)

### 1. Syntax & Lexical Structure (CRITICAL)

- [`syn-brackets-and-balance`](rules/syn-brackets-and-balance.md) - `()` and `{}` read as plain lists, `[a b]` reads as the list literal `(list a b)`; keep every opener matched, and know the balance check only catches net imbalance.
- [`syn-int-literal-range`](rules/syn-int-literal-range.md) **[CRITICAL]** - Write any constant at or above 2^63 as its negative two's-complement `Int`; never write an unsigned-sized literal.
- [`syn-keywords-and-symbols`](rules/syn-keywords-and-symbols.md) - Use `:keyword` tokens only where a form expects them; they are not values.
- [`syn-naming-conventions`](rules/syn-naming-conventions.md) - kebab-case functions and variables, PascalCase types and variants, `?` predicates, `_` for unused, two-space indent; treat special-form names as reserved.
- [`syn-no-stray-characters`](rules/syn-no-stray-characters.md) **[CRITICAL]** - Never write `@`, `#`, `$`, `|`, `^`, `\`, a `.` that does not start `.name`, or non-ASCII bytes outside strings and comments: each is a located `E_INVALID_CHAR`. `'`, `` ` ``, `,` and `,@` are reader syntax, and `,`/`,@` outside a quasiquote or macro template is `E_MALFORMED_FORM`.
- [`syn-string-literals`](rules/syn-string-literals.md) - Use only the supported escapes (`\n \t \r \0 \" \\ \e \xNN`); remember strings are NUL-terminated byte pointers.
- [`syn-list-literals-and-quote`](rules/syn-list-literals-and-quote.md) **[CRITICAL]** - Build a `List` with `[a b c]` or `(list a b c)`, quote constant data with `'(1 2 3)`, and fill holes with `` `(... ,x ,@xs) ``; every element has one type, and a name inside quoted data is `E_MALFORMED_FORM`.

### 2. Functions, Bindings & Control Flow (CRITICAL)

- [`fn-types-drive-codegen`](rules/fn-types-drive-codegen.md) **[CRITICAL]** - Let inference type your values: `print`, `=`/`<` and arithmetic follow the static type the checker proved for every expression; a program whose types conflict or cannot be determined does not compile, so there is no untyped fallback to guard against.
- [`fn-begin-multi-form-bodies`](rules/fn-begin-multi-form-bodies.md) **[CRITICAL]** - Wrap every multi-step `if` branch and `match` arm body in `begin`; other bodies (`defn`, `let`, `fn`, `while`, `for`, `cond` clauses, `catch` handlers) already sequence their forms, while `test` and `defmacro` take exactly one.
- [`fn-conditionals`](rules/fn-conditionals.md) - Conditions must be `Bool`; an `if` without else and a `cond` without `else` are `Unit`, so give both a final branch whenever their value is used.
- [`fn-for-has-no-step`](rules/fn-for-has-no-step.md) - `for` has no step clause: the body must `set!` the loop variable, or the loop never ends.
- [`fn-int-float-separation`](rules/fn-int-float-separation.md) - Never mix `Int` and `Float` in one operation: it is `E_TYPE_MISMATCH`, there is no implicit conversion, and there are no conversion built-ins.
- [`fn-integer-arith-unchecked`](rules/fn-integer-arith-unchecked.md) **[CRITICAL]** - Guard divisors and overflow yourself: compiled integer arithmetic wraps silently, and division by zero kills the process with SIGFPE, which `try` cannot catch and which discards buffered output.
- [`fn-let-single-binding`](rules/fn-let-single-binding.md) - `let` binds exactly one name; nest `let`s for several, and prefer the bare `(let name value body...)` spelling.
- [`fn-main-and-exit-status`](rules/fn-main-and-exit-status.md) - Every executable needs `(defn main () ...)` of type `() -> Int`: no parameters, and a body that ends in an Int, which becomes the exit status. End it with an explicit `0`.
- [`fn-no-named-let-or-early-return`](rules/fn-no-named-let-or-early-return.md) - There is no `return`, named `let` or `let*`: structure code as small tail-recursive helpers with accumulators (tail calls, direct or through a function value, are jumps), or `while` loops.
- [`fn-no-return-type-slot`](rules/fn-no-return-type-slot.md) - Parameters are a bare name or `(name Type)`; there is no return-type annotation and no annotation on `let`.
- [`fn-toplevel-def`](rules/fn-toplevel-def.md) - Use a top-level `(def name expr)` for constants: an immutable global, evaluated once in source order before `main` or the tests.
- [`fn-print-semantics`](rules/fn-print-semantics.md) - `print` writes each argument on its own line and returns `Unit`; build one-line output with `str-concat` and `Show.show`.
- [`fn-string-equality`](rules/fn-string-equality.md) - `=`/`==`/`!=` on Strings compare contents and `<`/`>`/`<=`/`>=` order them by bytes, everywhere: every expression has a static type, so there is no address-comparison fallback. Comparing a String with anything else is `E_TYPE_MISMATCH`.
- [`fn-underscore-discard`](rules/fn-underscore-discard.md) - Use `_` (or a `_`-prefixed name) for anything deliberately unused; never invent dummy names.
- [`fn-unlowered-forms`](rules/fn-unlowered-forms.md) **[CRITICAL]** - Do not use forms that type-check but are not lowered: `read-line`, `exit`, `close`, `with-resource` cleanup, and `alias`; `make-struct` and `make-variant` no longer compile at all.

### 3. Structs, ADTs & Collections (CRITICAL)

- [`data-adt-declaration`](rules/data-adt-declaration.md) - Declare sum types with `(deftype Name (Variant FieldType...) ...)`; an unknown uppercase field type is a type parameter.
- [`data-collections-persistent`](rules/data-collections-persistent.md) **[CRITICAL]** - `Vec` is generic (`(Vec T)`), `core/map` is `(Map String V)`, and `collections/map`/`collections/set` hold `Int`s; every operation returns an updated value: always rebind to the result and treat the old value as used up, because versions share storage.
- [`data-equality-structural`](rules/data-equality-structural.md) **[CRITICAL]** - `==`/`!=` on structs and ADTs compare structurally by content through a generated `T.==`; `<`/`>` on them are type errors, so order records with a derived `Ord.compare` or by hand.
- [`data-field-types`](rules/data-field-types.md) - Declare field types on `deftype` variants and `defstruct` fields: construction is checked against them, a pattern-bound name or field read carries the declared type, and an untyped field makes the record generic in that field.
- [`data-no-redeclare-prelude`](rules/data-no-redeclare-prelude.md) - Never redeclare `Option`, `Result`, `List`, their constructors, or any prelude function name; pick another name.
- [`data-no-tuples-implicit-generic-structs`](rules/data-no-tuples-implicit-generic-structs.md) - Use a struct or a single-variant ADT where you want a tuple; make a struct generic by leaving fields untyped or typing them with uppercase names; don't rely on `alias`.
- [`data-reconstruct-field-order`](rules/data-reconstruct-field-order.md) **[CRITICAL]** - When rebuilding a struct or variant, pass every field in declaration order — double-check against the `defstruct`/`deftype`.
- [`data-struct-basics`](rules/data-struct-basics.md) - Declare with `defstruct`, build with `make-Name`, read with `v.field` (chains: `v.a.b`; any expression: `(expr).field`) or `(struct-get v "field")` with a literal field name.
- [`data-struct-immutable-rebind`](rules/data-struct-immutable-rebind.md) - Struct fields never change: to "update" one, build a new struct and rebind a `let-mut` name to it.
- [`data-two-map-types`](rules/data-two-map-types.md) - Pick one map per program: `core/map` (String keys, any values, `Option` results) or `collections/map` (Int keys and values, a default value, arena-backed).
- [`data-unique-variant-names`](rules/data-unique-variant-names.md) **[CRITICAL]** - Give every variant a name unique across all types in the program, and declare each type name exactly once.
- [`data-views-and-slices`](rules/data-views-and-slices.md) - Parse and window data without copying: `text/view` gives `StrView` (a substring) and `Cursor` (a parsing position), `collections/slice` gives `Slice` (a window on a Vec's storage); copy only at the end with `view-to-string` or `slice-to-vec`.

### 4. Pattern Matching (CRITICAL)

- [`match-arm-complex`](rules/match-arm-complex.md) - In a match arm, never combine a constant with two or more calls in one arithmetic expression; bind the calls with `let` first.
- [`match-arm-shape`](rules/match-arm-shape.md) - Write arms as `(Variant binder... body)` with exactly one binder (a name or `_`) per field; the grouped `((Variant binder...) body)` form means the same.
- [`match-catch-all-last`](rules/match-catch-all-last.md) - Put `_` last, exactly once, with no guard; an arm after a catch-all is `E_UNREACHABLE_MATCH_ARM`.
- [`match-exhaustive-or-underscore`](rules/match-exhaustive-or-underscore.md) - Cover every variant, or end with a single `_` arm; a non-exhaustive constructor match is a compile-time error.
- [`match-guards-literal-arms-only`](rules/match-guards-literal-arms-only.md) **[CRITICAL]** - Use `(when cond)` guards only after plain literal alternatives; test constructor fields inside the arm body instead.
- [`match-literal-requires-underscore`](rules/match-literal-requires-underscore.md) - Literal, OR and range matches must end with `_`, bind nothing, and cannot be mixed with constructor arms.
- [`match-misspelled-last-arm`](rules/match-misspelled-last-arm.md) **[CRITICAL]** - Spell every constructor in a match arm exactly: an arm head that is not a known constructor is a catch-all that binds nothing.
- [`match-no-nested-patterns`](rules/match-no-nested-patterns.md) **[CRITICAL]** - Match one constructor level at a time: a field position holds only a name or `_`; a nested pattern is `E_NESTED_PATTERN`.

### 5. Error Handling (CRITICAL)

- [`err-helper-argument-order`](rules/err-helper-argument-order.md) - Option/Result helpers take the value first and the function second; `collections/collections` list helpers take the collection **last**; `-unwrap` helpers require a default.
- [`err-no-assert-unwrap`](rules/err-no-assert-unwrap.md) - `assert` takes a `Bool` and shows a string-literal message (`PANIC: msg`), otherwise a fixed `assert failed`; `unwrap` takes only an `Option` and panics with `unwrap on None`. Use literal messages and the `-expect` helpers where the message matters.
- [`err-result-for-expected-failures`](rules/err-result-for-expected-failures.md) - Return `Result` (`Ok`/`Err`) for failures a caller can handle, `Option` (`Some`/`None`) for absence; reserve `error` for unrecoverable conditions.
- [`err-try-catches-error-not-err`](rules/err-try-catches-error-not-err.md) **[CRITICAL]** - `try`/`catch` intercepts `error` panics only; an `(Err ...)` value passes straight through it.
- [`err-try-any-arity`](rules/err-try-any-arity.md) - Catching `error` from a call of any arity works now (the 2/4-argument hang was fixed 2026-09-24); remove old workarounds that kept erroring functions at odd arity.

### 6. Contracts (HIGH)

- [`contract-checks-and-profiles`](rules/contract-checks-and-profiles.md) - Use `requires`/`ensures`/`invariant` for checked contracts (`E_CONTRACT_VIOLATION`, `result` in `ensures`), pick a profile with `--contracts=P` or `(contracts P)`, recover by error code with `recover`, and roll back `let-mut` state with `checkpoint`.


## Tier 2 — The Language Model

### 7. Ownership, Capabilities & Regions (HIGH)

- [`own-capability-kinds`](rules/own-capability-kinds.md) **[CRITICAL]** - Know that capabilities are not part of the type checker: `TCap`/`TMut` are decided by binding form (`let` versus `let-mut`/`for`) in `mutability_check`, `Secret` by taint in `secret_check`, and the only capability-like type is `(Pin a)`, the result of `ffi-pin`.
- [`own-heap-never-freed`](rules/own-heap-never-freed.md) - Values that escape into the heap live until process exit; per-call regions reclaim everything the compiler proves short-lived and the reuse pass updates unique dead values in place, so for bulk temporaries use `with-region`, or an `allocator/allocator` arena for raw memory.
- [`own-let-copies-word`](rules/own-let-copies-word.md) - Know that every value is one 64-bit word and `let` copies the word: scalars become independent copies, pointers become aliases.
- [`own-let-mut-only-set`](rules/own-let-mut-only-set.md) - `set!` only a plain name bound by `let-mut` (or a `for` variable) in the current scope; everything else is immutable.
- [`own-no-closure-captured-mutation`](rules/own-no-closure-captured-mutation.md) **[CRITICAL]** - Never `set!` a captured variable inside a closure; have the closure return a new value and rebind it in the owning scope.
- [`own-regions-status`](rules/own-regions-status.md) - Regions are real: per-call frame and result regions, the process heap, `with-region` scopes and Pin; Global and Circular are names only.
- [`own-stack-promotion`](rules/own-stack-promotion.md) - Keep short-lived values from escaping and the compiler places them in the call's frame region; a `let`-bound ADT that is only matched or printed goes further, into the frame itself.
- [`own-with-region`](rules/own-with-region.md) - Wrap bulk temporary work in `(with-region (arena ...) body)` or `(with-region (fixed ...) body)` to release it when `body` ends and cap its memory; return only values built outside the region.

### 8. Type System (CRITICAL)

- [`type-annotations-constrain`](rules/type-annotations-constrain.md) - Annotate a parameter as `(name Type)` to fix its type: it unifies with the parameter everywhere (top-level functions and lambdas alike), so any clash is `E_TYPE_MISMATCH`; annotations are optional where use already determines the type.
- [`type-sound-checking`](rules/type-sound-checking.md) **[CRITICAL]** - Write well-typed code: type checking is sound and strict, so every type error in the program is reported (`E_TYPE_MISMATCH`, `E_INFINITE_TYPE`, `E_CANNOT_INFER`, `E_UNBOUND_VARIABLE`) and then the compile fails.
- [`type-value-representation`](rules/type-value-representation.md) - Reason about values as single 64-bit words with a fixed layout; the type checker, not the word, says what a value is.

### 9. Generics (HIGH)

- [`gen-generic-adts`](rules/gen-generic-adts.md) - Make data generic with ADTs (or structs) whose field types are unknown **uppercase** names; each name is one type parameter, checked at every construction and use.
- [`gen-monomorphization-naming`](rules/gen-monomorphization-naming.md) - Know the per-type function names: impl methods are `Trait.method_<type key>`, and a trait-generic function's instances are `<key>~T1,T2` (argument types in order, fully spelled); more than 256 instances of one function is `E_CANNOT_INFER`.
- [`gen-no-type-parameter-syntax`](rules/gen-no-type-parameter-syntax.md) - Never write the spec's type-parameter groups `((T) x)` or `((T : Ord) a b)`; leave parameters unannotated, or annotate them with an uppercase type variable such as `(a T)`.
- [`gen-operators-follow-types`](rules/gen-operators-follow-types.md) - Use `==`, `<` and arithmetic directly in generic code: each instance uses its own types' operators (Strings compare by content, `<` on Strings is byte order, Floats use SSE); for records, `==` is structural and ordering goes through `Ord.compare` or a comparator argument.
- [`gen-unannotated-is-polymorphic`](rules/gen-unannotated-is-polymorphic.md) - Write generic functions by leaving parameters unannotated; top-level functions get polymorphic types (let-polymorphism) and each call instantiates them.
- [`gen-per-type-instances`](rules/gen-per-type-instances.md) **[CRITICAL]** - Write generic code freely: a function whose body prints, compares, does arithmetic on, or calls a trait method on a type parameter is compiled once per concrete argument-type tuple, at every call and every use as a value; nothing is dispatched at run time.

### 10. Traits (HIGH)

- [`trait-qualified-calls`](rules/trait-qualified-calls.md) - Call trait methods with dot syntax, `(r.method args...)` or `((expr).method args...)`, picked by the receiver's type; use the qualified `(Trait.method r args...)` when several traits share the name and the receiver's type is unknown.
- [`trait-coherence-and-orphans`](rules/trait-coherence-and-orphans.md) - Declare the trait (`(trait Name ...)`) in your package before implementing it for a type you don't own, write each `(Trait, Type)` impl exactly once, and use `(impl-not Trait Target)` to forbid a pair.
- [`trait-derive-show`](rules/trait-derive-show.md) - Derive Show, Debug, Eq, Ord, Hash and Clone with `(derive T ...)`; every field type must implement the trait.
- [`trait-static-dispatch`](rules/trait-static-dispatch.md) **[CRITICAL]** - Implement traits for any type — structs, multi-variant ADTs, primitives, generic types — and call them only where the receiver's type is known at compile time: every trait call resolves statically, and there is no run-time dispatch.
- [`trait-no-dyn-use-adt-wrapper`](rules/trait-no-dyn-use-adt-wrapper.md) - Model open "trait object" designs with an ADT wrapper, a list of structs, or function values; there is no `dyn`.

### 11. Closures (HIGH)

- [`closure-capture-by-value`](rules/closure-capture-by-value.md) - Expect a closure to see each captured variable's value at the moment the closure was created.
- [`closure-explicit-fn-syntax`](rules/closure-explicit-fn-syntax.md) - Write anonymous functions as `(fn (params) body)` or `(lambda (params) body)`; there is no shorthand.
- [`closure-no-recursive-lambda`](rules/closure-no-recursive-lambda.md) - Write recursive helpers as top-level `defn`s; a lambda cannot refer to itself.

### 12. Macros (HIGH)

- [`macro-args-spliced`](rules/macro-args-spliced.md) - Use each parameter once in a macro body, or bind it with `let` inside the body so the argument is evaluated exactly once.
- [`macro-hygiene`](rules/macro-hygiene.md) - Pass everything a macro needs from the call site as an argument; the body's binders are renamed and it cannot see the caller's locals.
- [`macro-name-positions`](rules/macro-name-positions.md) - Pass an identifier when a macro parameter lands in a name position (binder, `set!` target, `defn`/`deftype`/`impl` name).
- [`macro-no-recursion`](rules/macro-no-recursion.md) - Never write a macro that can expand into a call of itself, even behind an `if`; put recursion in functions.
- [`macro-prefer-functions`](rules/macro-prefer-functions.md) - Use a macro only when you need call-by-name evaluation (skip or repeat an argument) or new binding syntax; otherwise write a function.
- [`macro-shadows-functions`](rules/macro-shadows-functions.md) - Remember that a macro takes over every call to a same-named imported function, program-wide.
- [`macro-template-is-literal-code`](rules/macro-template-is-literal-code.md) **[CRITICAL]** - Write the macro body as the literal expansion, in exactly one form: parameters are replaced by argument source, and a backquote in a template builds list data, not code.
- [`macro-top-level-and-registration`](rules/macro-top-level-and-registration.md) - Define macros only at top level, once per name; they may be used before their definition.
- [`macro-quasiquote-and-rest`](rules/macro-quasiquote-and-rest.md) **[CRITICAL]** - Take a variable number of macro arguments with `&rest name` and splice them with `,@name` only where a form takes any number of expressions; to build a list from them, use `` `(,@name) ``, never `(list ,@name)` or `[... ,@name]`.


## Tier 3 — Systems Programming

### 13. FFI (CRITICAL)

- [`ffi-extern-word-sized-types`](rules/ffi-extern-word-sized-types.md) **[CRITICAL]** - Give every foreign function an `(extern ...)` signature built from word-sized types (Int, Bool, String, `Ptr`, `(Pin a)`, handles, `(Fn (A ...) R)`), declare the C side with `int64_t` and pointers, and cross a `Float` only as its bits.
- [`ffi-linking`](rules/ffi-linking.md) - Link your own C either with a package `native` block (preferred) or by emitting assembly and running `cc` yourself; the single-file CLI takes no extra objects or libraries.
- [`ffi-memory-across-boundary`](rules/ffi-memory-across-boundary.md) - Declare C-owned pointers as `Ptr`, copy their bytes into a Zyl string with `alloc-cstr` before freeing them, and give C writable buffers from `alloc-malloc` or an arena.
- [`ffi-pin-passes-pointer`](rules/ffi-pin-passes-pointer.md) **[CRITICAL]** - Use `ffi-pin` only when C expects a **pointer to** a value (out-parameters): it gives a `(Pin a)`, the address of a slot, which an extern must declare as `(Pin a)`. Pass Ints and Strings directly.
- [`ffi-timeout-always-last`](rules/ffi-timeout-always-last.md) **[CRITICAL]** - Always end `ffi-call` with a positive timeout literal in milliseconds: `(ffi-call "sym" args... 1000)`. It is checked at compile time and enforced at run time.
- [`ffi-wrap-each-call`](rules/ffi-wrap-each-call.md) - Put each foreign function's `extern` next to one small, named Zyl wrapper, so its signature, timeout, pinning and pointer conversion live in one place.
- [`ffi-extern-required`](rules/ffi-extern-required.md) **[CRITICAL]** - Declare every foreign symbol with `(extern "sym" (ParamType ...) ResultType)` before the program can `ffi-call` it; call `zyl_*` runtime entries without one, and never the raw ones.

### 14. Actors (CRITICAL)

- [`actor-always-wait`](rules/actor-always-wait.md) **[CRITICAL]** - Know how actors end: returning from `main` drains every actor, while `actor-wait` stops one at once and drops the messages still queued for it. Wait explicitly only where you need an ordering point.
- [`actor-no-closure-messages`](rules/actor-no-closure-messages.md) **[CRITICAL]** - Use `send` + `(receive)` for every message: the runtime's closure messages (`zyl_actor_send_closure`) cannot be sent from a Zyl program.
- [`actor-limits`](rules/actor-limits.md) - Design within the runtime's limits: at most 1024 actors per process (ids never reused), ~8 MB actor stacks, unbounded mailboxes, and a panic in any actor kills the whole process.
- [`actor-no-let-mut-crossing`](rules/actor-no-let-mut-crossing.md) - Never reference a `let-mut` variable in a `send` message or spawned closure; snapshot it with `let` first.
- [`actor-output-nondeterministic`](rules/actor-output-nondeterministic.md) - Let exactly one actor (usually `main`) produce ordered output, or collect results and print after `zyl_actor_wait_all`.
- [`actor-send-is-discarded`](rules/actor-send-is-discarded.md) **[CRITICAL]** - Exchange data messages with `send` + `(receive)`, and reply to `(actor-self)` ids (main included); a sent message is dropped only if the target never calls `receive`, and `receive`'s result is not type-checked.
- [`actor-spawn-zero-arg-entry`](rules/actor-spawn-zero-arg-entry.md) **[CRITICAL]** - Pass `spawn` a named zero-argument function or a zero-parameter `fn`; captures of immutable values work (e.g. an `(actor-self)` id to reply to), an entry with a parameter is `E_TYPE_MISMATCH`, and a `let-mut` capture is `E_CAPABILITY_LEAK`.

### 15. Bits, Bytes & Buffers (HIGH)

- [`bits-atomics-aligned`](rules/bits-atomics-aligned.md) - Use `bytebuf-atomic-*` only at 8-aligned offsets that fit in the buffer, and know which ones return old vs new values.
- [`bits-bounds-fail-closed`](rules/bits-bounds-fail-closed.md) - Check the results of byte loads, stores and appends: out-of-range accesses do nothing and return 0 instead of failing.
- [`bits-bytebuf-basics`](rules/bits-bytebuf-basics.md) **[CRITICAL]** - Allocate packed bytes with `(bytebuf Region N)` using literal region and capacity; pass the handle as a `ByteBuf` or `ByteSlice`, annotating parameters the checker cannot settle.
- [`bits-defined-shift-counts`](rules/bits-defined-shift-counts.md) - Rely on Zyl's defined shift semantics (logical shifts by ≥64 give 0; `ashr` saturates), not on x86's mod-64 masking.
- [`bits-wide-loads-and-stores`](rules/bits-wide-loads-and-stores.md) - Use `load-u16`..`load-u64`, `load-i16`..`load-i64` and `store-*` for wide values, with an explicit `:le`/`:be`; buffers are typed `ByteBuf`/`ByteSlice`.
- [`bits-bytebuf-regions`](rules/bits-bytebuf-regions.md) - Write `Pin` for any buffer whose address you take or share; `Stack` buffers live in the frame region and must not escape, and the other bytebuf region rules are still unchecked.
- [`bits-shr-vs-ashr`](rules/bits-shr-vs-ashr.md) **[CRITICAL]** - Use `shr` (logical, zero fill) for bit patterns and `ashr` (arithmetic, sign fill) for signed numbers.

### 16. Secrets & Constant-Time (CRITICAL)

- [`secret-annotate-params`](rules/secret-annotate-params.md) - Mark key material with a `Secret` parameter annotation — `(k Secret)`, `(k (Secret Int))` or `(k (Secret Words))` — at every function that handles it.
- [`secret-branchless-masks`](rules/secret-branchless-masks.md) - Make secret-dependent decisions with masks from `math/secret/secret` (`ct-select`, `ct-eq`, `ct-is-zero`, `ct-mask`), never with `if`.
- [`secret-ct-eq-words-for-bytes`](rules/secret-ct-eq-words-for-bytes.md) **[CRITICAL]** - Compare MACs, tags, hashes and passwords with `ct-eq-words` / `ct-eq-words-bool`, never with `=` or a loop that stops at the first difference.
- [`secret-declassify-explicitly`](rules/secret-declassify-explicitly.md) - Make a secret-derived value public only through `declassify`, `ct-eq-bool` or `ct-eq-words-bool`, with a comment saying why it is safe.
- [`secret-five-prohibitions`](rules/secret-five-prohibitions.md) **[CRITICAL]** - Never branch on, index with, divide by, print, or send a `Secret`, and pass it to C only through `ffi-pin`.
- [`secret-unannotated-helpers-launder`](rules/secret-unannotated-helpers-launder.md) **[CRITICAL]** - Annotate every helper a secret flows through; an unannotated helper silently launders taint.
- [`secret-zeroize`](rules/secret-zeroize.md) - Frames that held secrets are zeroed on return; erase heap key material explicitly with `(zeroize words n)`, `(zeroize-bytes addr n)` or `(k.wipe)`.

### 17. Cryptography Library (MEDIUM)

- [`crypto-choose-primitives`](rules/crypto-choose-primitives.md) - Default to ChaCha20-Poly1305 for AEAD, X25519 + HKDF for key agreement, Ed25519 for signatures, `sysrng` for key material; never use `chacharng` for keys.
- [`crypto-representations`](rules/crypto-representations.md) - In `stdlib/math`, byte strings are `Words` arrays with one byte (0..255) per word, big numbers are 24-bit limbs least-significant first, and every entry point takes an `Arena` first.


## Tier 4 — Engineering & Tooling

### 18. Testing (HIGH)

- [`test-assert-equal-semantics`](rules/test-assert-equal-semantics.md) **[CRITICAL]** - `assert-equal` is typed: both sides must have one type (else `E_TYPE_MISMATCH`), and it compares by that type, by content for strings and records; only records with a `Secret` field fall back to a shallow comparison.
- [`test-compiler-internals`](rules/test-compiler-internals.md) - Test compiler passes by `use`ing the compiler modules and asserting on their results, and pin every compile-fail test to its code with `; expect-error: CODE`.
- [`test-program-library-tests-split`](rules/test-program-library-tests-split.md) - Structure a project as a library module (no `main`), a program file (`use` + `main`), and a test file (`use` + tests + `run-tests`).
- [`test-read-summary-line`](rules/test-read-summary-line.md) - Judge a test run by its output (`FAIL` lines and the `test result:` summary), not by the exit status.
- [`test-regression-runner`](rules/test-regression-runner.md) - Run `./run_regression_tests.sh --full --no-boot --filter <word>` for targeted checks; `--filter` narrows the selected mode, it does not select one.
- [`test-toplevel-forms-and-run-tests`](rules/test-toplevel-forms-and-run-tests.md) - Write tests as flat top-level `(test "name" body)` forms, end the file with `(run-tests)`, and do not define `main` in a test file.
- [`test-unimplemented-features`](rules/test-unimplemented-features.md) **[CRITICAL]** - Don't use `test-suite`, `setup`/`teardown`, `test-property`, `test-compile`, `assert-fail`, keyword options or the `run-tests-*` helpers: they compile and do nothing, silently drop tests, or are rejected.

### 19. Modules & Packages (HIGH)

- [`pkg-canonical-keys`](rules/pkg-canonical-keys.md) - Read linker symbols and resolver messages as canonical keys `<package>@<major>::<module>::<symbol>`, mangled injectively.
- [`pkg-capabilities`](rules/pkg-capabilities.md) - Declare the narrowest `(capabilities ...)` set a package needs; know that `main`, top-level tests and manifest-less files are not checked.
- [`pkg-features`](rules/pkg-features.md) - Use features only to **add** top-level definitions or impls via top-level `(feature-gate f def)`; never to replace one.
- [`pkg-library-no-main`](rules/pkg-library-no-main.md) **[CRITICAL]** - Never define `main` (or other generic names an importer might define) in a module meant to be `use`d.
- [`pkg-manifest`](rules/pkg-manifest.md) - Write `zyl.pkg` with `name`, `version`, `zyl` and `edition`, scoped names, strict SemVer, and **bare minimum** version requirements.
- [`pkg-modules-and-use`](rules/pkg-modules-and-use.md) - A module is a `.zyl` file named by its path; import with `(use path)`, `(use pkg)`, `(use pkg:module)`, optionally with a whitespace-separated `{ ... }` list.
- [`pkg-mvs-lock-store`](rules/pkg-mvs-lock-store.md) - Upgrade by editing a requirement; commit `zyl.lock`; build CI with `zyl build --locked`; run `zyl fetch` (the only networked command) before building registry deps.
- [`pkg-native-dependencies`](rules/pkg-native-dependencies.md) - Ship C code declaratively with `(native (sources ...) (cflags ...) (include-dirs ...) (link-libs ...))`; build scripts are forbidden.
- [`pkg-pub-and-explicit-imports`](rules/pkg-pub-and-explicit-imports.md) - Mark the public surface with inline `(pub defn ...)` and import with explicit `{ ... }` lists.
- [`pkg-stdlib-resolution`](rules/pkg-stdlib-resolution.md) **[CRITICAL]** - When a stdlib or compiler change "doesn't take effect", suspect a stale `~/.zyl`: set `ZYL_HOME=$PWD/build/boot`, or refresh the install with `./uninstall.sh && ./install.sh` (a verified `./boot.sh` does this for you).
- [`pkg-use-what-you-construct`](rules/pkg-use-what-you-construct.md) **[CRITICAL]** - Explicitly `use` every module whose types, constructors or functions your module relies on, even if they "happen to be visible".

### 20. Determinism (HIGH)

- [`det-left-to-right`](rules/det-left-to-right.md) **[CRITICAL]** - Rely on strict left-to-right evaluation everywhere, and never write code (or compiler passes) that reorders side effects.
- [`det-no-address-dependent-output`](rules/det-no-address-dependent-output.md) - Never let an address influence output: give every printed type a `Show`, don't compare handles expecting content, don't derive names or ordering from pointers.
- [`det-nondeterminism-sources`](rules/det-nondeterminism-sources.md) - Keep clock, environment, PID, kernel entropy and multi-actor output out of anything that must be reproducible — the compiler will not warn you.
- [`det-ordered-collections`](rules/det-ordered-collections.md) - Iterate only ordered structures (lists, association lists, insertion-ordered arrays); a hash table may be probed by key but never iterated.

### 21. Tooling (CLI, REPL, LSP) (MEDIUM)

- [`tool-cli-arguments`](rules/tool-cli-arguments.md) - Put the source file first: `zyl file.zyl [-o out] [--emit-asm]`; everything else is a subcommand, and unknown words become the output path.
- [`tool-debugging-the-pipeline`](rules/tool-debugging-the-pipeline.md) **[CRITICAL]** - Debug with `--emit-asm`, the `ZYL_*` bisection switches (`ZYL_MIR=0`, `ZYL_INLINE=0`, `ZYL_REUSE=0`, `ZYL_REGIONS=0`), `ZYL_DEBUG_STAGES`/`ZYL_DEBUG_TYPES`, `zyl eval` differential runs, and small driver programs that `use` compiler modules.
- [`tool-eval-differential`](rules/tool-eval-differential.md) - Use `zyl eval` / the REPL for fast iteration, but confirm behavior with a compiled binary: the interpreter differs on actors, FFI, division by zero and speed.
- [`tool-lsp-and-editors`](rules/tool-lsp-and-editors.md) **[CRITICAL]** - Point any LSP client at `zyl-lsp` for the compiler's own diagnostics, type errors included; expect the pre-type checks one at a time, every type error at once, no capability check, byte-based columns and name-based (not scope-based) navigation.
- [`tool-repl`](rules/tool-repl.md) - Use the REPL (`zyl repl`) to explore expressions and definitions; each name can be defined once per session.

### 22. Project Idioms (MEDIUM)

- [`proj-buf-append-appends`](rules/proj-buf-append-appends.md) **[CRITICAL]** - Use `buf-append` only on a fresh `StrBuf` from `buf-new` or one you intend to extend: it appends after what the buffer already holds, it does not copy.
- [`proj-file-io`](rules/proj-file-io.md) **[CRITICAL]** - Use the built-in `file-open`/`file-read`/`file-write`/`file-close` forms with a literal mode and String data; check the descriptor and read with an explicit maximum size.
- [`proj-idioms`](rules/proj-idioms.md) - Write Zyl in its native style: small pure functions, recursion with accumulators, ADTs plus exhaustive `match`, state threaded through return values, zero-copy views for parsing, short `let` chains, I/O at the edges.


## Tier 5 — Compiler Contributors (`stdlib/compiler/`, `selfhost/`)

### 23. Bootstrap & Fixed Point (CRITICAL)

- [`boot-match-arm-call-sums`](rules/boot-match-arm-call-sums.md) - In a match arm, bind call results with `let` before combining a constant with two or more calls in one arithmetic expression (`E_MATCH_ARM_COMPLEX`); elsewhere, calls may be combined directly.
- [`boot-field-parity-lifted`](rules/boot-field-parity-lifted.md) - Do not pad record types to an even field count: odd counts in nested constructions compile correctly in both backends. The even `CheckState` in `sexp_balance.zyl` is a leftover workaround, not a rule.
- [`boot-failure-modes`](rules/boot-failure-modes.md) - Map a bootstrap failure to its cause before changing anything.
- [`boot-fixed-point-workflow`](rules/boot-fixed-point-workflow.md) **[CRITICAL]** - After editing `stdlib/compiler/*.zyl`, `selfhost/` or `runtime/actor_runtime.c`: reseed, verify, commit the seed.
- [`boot-lifted-constraints`](rules/boot-lifted-constraints.md) - Know which historical bootstrap constraints are lifted, so you neither follow dead rules blindly nor reintroduce the patterns they guarded against.
- [`boot-moderate-bodies`](rules/boot-moderate-bodies.md) - Keep compiler function bodies moderate and flat; prefer short `let` chains and helpers over deep nesting.
- [`boot-module-build`](rules/boot-module-build.md) - The compiler is built like any program: every boot stage compiles `selfhost/driver.zyl`, and module resolution follows its `(use ...)` tree through `stdlib/`. A compiler module is reached only through a `use`.
- [`boot-one-deftype-per-name`](rules/boot-one-deftype-per-name.md) **[CRITICAL]** - Define each type name exactly once across everything one program imports, and keep variant names unique.
- [`boot-parens-per-file`](rules/boot-parens-per-file.md) **[CRITICAL]** - Keep every top-level form independently balanced; a missing closer swallows everything after it in the same file.
- [`boot-two-step-syntax`](rules/boot-two-step-syntax.md) - Introduce new syntax (and any new runtime function the compiler calls) in two steps: teach the compiler to accept it and reseed, then start using it in the compiler's own source.

### 24. ICNF (MEDIUM)

- [`icnf-lowering-map`](rules/icnf-lowering-map.md) - Know what each source form lowers to, so you can predict codegen and write passes over ICNF.
- [`icnf-new-form-needs-case`](rules/icnf-new-form-needs-case.md) **[CRITICAL]** - Give every new special form a case in the type pass and in `ic-expr-node`, and every new `Icnf` node a case in each ICNF pass and in both backends: lowering and the native backend's `ml-expr` turn anything they do not recognize into 0 silently.
- [`icnf-optimizer-scope`](rules/icnf-optimizer-scope.md) **[CRITICAL]** - Expect four safe ICNF optimizations (small-function inlining, copy propagation of let-bound locals, integer constant folding for opcodes 0–10, dead-branch elimination), then in-place reuse after region inference; never add an optimization that reorders or drops effects.
- [`icnf-regions-are-a-rewrite`](rules/icnf-regions-are-a-rewrite.md) - Region inference is two ICNF passes, the conservative `IStackVariant` rewrite and the whole-program `rg-*` classification into the `icnf-regions` side table; every extension must fail toward the heap.
- [`icnf-tree-structure`](rules/icnf-tree-structure.md) - Treat ICNF as an untyped structured tree of 21 `Icnf` constructors (not SSA), with three parameter representation kinds and every analysis result in side tables keyed by node.
- [`icnf-reuse-pass`](rules/icnf-reuse-pass.md) - In-place reuse (`reuse.zyl`, after region inference) lets a construction take the block of a value that is provably unique and dead; it only records decisions (`icnf-reuse`, owning clones `f~own`), the native backend acts on them, and every extension must fail toward allocating.

### 25. x86_64 Codegen (HIGH)

- [`cg-c-call-alignment`](rules/cg-c-call-alignment.md) **[CRITICAL]** - Keep `rsp` 16-byte aligned at every call into C: the stack machine realigns around each C call (`cg-ext-call-aligned`); the native backend aligns its frame once, in the prologue, and only when the function calls C.
- [`cg-call-arg-staging`](rules/cg-call-arg-staging.md) - Evaluate call arguments strictly left to right and never let one argument's evaluation clobber another's value: the stack machine stages them in scratch slots (or takes its direct-register path when that is provably safe), the native backend evaluates them into fresh vregs and moves them into argument registers with one parallel move.
- [`cg-callee-saved-registers`](rules/cg-callee-saved-registers.md) **[CRITICAL]** - Every generated function preserves each callee-saved register it touches: the stack machine uses and saves only `rbx` and `r12`; the native backend allocates `rbx` and `r12`–`r15` and saves exactly the ones it used. Hand-written sequences use only the scratch registers.
- [`cg-closure-call-protocol`](rules/cg-closure-call-protocol.md) - Calls through a local holding a function value go through the stack machine's `cg-call-indirect`, which passes one extra trailing argument: the closure env, or 0 for a plain code address. Every function either backend emits must start with `push rbp`, because the tag test relies on it.
- [`cg-emission-appends`](rules/cg-emission-appends.md) - Emit assembly by appending to the text buffer (`zyl_str_append_capped` via `cg-emit*`), never by copying.
- [`cg-kind-of`](rules/cg-kind-of.md) - Remember codegen decides print format, float arithmetic and string/variant comparison from `kind-of` (0 word, 1 String, 2 Float, 3 variant): the literal/shape kind first, else the kind the type pass recorded for the ICNF node (`icnf-kinds`); and any non-word operator keeps a function out of the native backend.
- [`cg-stack-machine-fallback`](rules/cg-stack-machine-fallback.md) - Know the stack machine as the fallback backend: every function the native backend declines (`mb-eligible` false, about 5% of the compiler's own functions, or all of them under `ZYL_MIR=0`) is compiled to code over `rax` with `rbp`-relative slots and no register allocator.
- [`cg-symbols-and-entry`](rules/cg-symbols-and-entry.md) - User functions are labelled by mangled canonical keys (`zy_...`), the user entry is `_ZYL_main`, and every program shares one C `main` stub that runs it on a huge stack.
- [`cg-native-backend-mir`](rules/cg-native-backend-mir.md) **[CRITICAL]** - Most functions are compiled by the native backend: ICNF lowered to MIR (`ml-*`), liveness and linear-scan register allocation (`mir.zyl`), emission (`mb-*`). Extend it by widening `ml-ok` and `ml-expr` together; anything `mb-eligible` rejects falls back to the stack machine.

### 26. Writing Compiler Passes (HIGH)

- [`pass-adding-a-pass`](rules/pass-adding-a-pass.md) - Add a compiler pass by writing the module, calling and `use`-ing it from `pipeline.zyl` at the right point in the phase order, testing it, and reseeding.
- [`pass-avoid-repeated-subtree-work`](rules/pass-avoid-repeated-subtree-work.md) - Visit each subtree once per pass; a pass that re-walks a subtree per visit goes exponential in nesting depth.
- [`pass-conservative-failure-direction`](rules/pass-conservative-failure-direction.md) **[CRITICAL]** - Make soundness checks fail closed (reject what they cannot prove), lint-like checks fail open (miss a diagnostic rather than reject valid code), and transformations fail safe (fall back to the always-correct path).
- [`pass-copy-spans`](rules/pass-copy-spans.md) - When a pass rebuilds an `Expr` node, copy the original's source span onto the replacement: `(ffi-call "zyl_span_copy" new-node old-node 1000)`.
- [`pass-keep-kinds`](rules/pass-keep-kinds.md) - When a pass rebuilds an ICNF node, carry its side-table facts across (`ic-keep-kind` for the kind, `opt-keep` for kind, span, scalar and ADT marks, plus `icnf-region-set` after region inference), and never read the type pass's tables for a node it did not see.
- [`pass-diagnostics`](rules/pass-diagnostics.md) - Report new errors through `err-at` (or `err-at-labels`) with a node and a catalogued code from `error_codes.zyl`, warnings through `err-warn-at`, and type errors through `ta-type-error`, so they print located with a help line, reach the LSP, and work in JSON mode.
- [`pass-instance-naming`](rules/pass-instance-naming.md) - Treat the type pass's canonical type text (`ta-canon`) as part of the fixed point: per-type instance names `f~T` are built from it, and there is no separate monomorphization pass any more.
- [`pass-no-allocation-in-lookups`](rules/pass-no-allocation-in-lookups.md) **[CRITICAL]** - Never allocate inside a comparison or lookup that runs per element of a table: in the compiler most such temporaries still land in the heap, which is never freed, so every one is kept for the whole compile.
- [`pass-no-stdout`](rules/pass-no-stdout.md) - Never write to stdout from a compiler pass; emit every warning and every reported error through `zyl_warn_emit`.
- [`pass-state-threading`](rules/pass-state-threading.md) - Thread state records through a pass and return small wrapper ADTs when a function yields a value plus new state; when a pass needs a table keyed by node or name instead, use a typed runtime table that is cleared at the start of each program and only probed by key.
- [`pass-string-eq-in-compiler`](rules/pass-string-eq-in-compiler.md) **[CRITICAL]** - In compiler source, compare strings with `str-eq` (a `Bool`) or `=` on String-typed operands; both compare contents. Never compare strings by address.
- [`pass-total-structural-match`](rules/pass-total-structural-match.md) - In tree-rewriting passes, list every constructor explicitly; use `_` only in read-only checks that care about a few forms.
- [`pass-evaluation-order-sets`](rules/pass-evaluation-order-sets.md) **[CRITICAL]** - When a lowering or codegen shortcut reads a local out of order (late, or straight into a register), guard it with `icnf-has-set` / `icnf-sets`: only a `set!` inside the expression itself can change a local while it is evaluated.

---

## How to Use

1. **Writing new code**: skim Tier 1 and the categories your task touches; apply every **[CRITICAL]** rule.
2. **Reviewing code**: walk [references/pitfalls.md](references/pitfalls.md) top to bottom.
3. **Hitting an error**: look the code up in [references/error-codes.md](references/error-codes.md); for odd behavior without an error, use [references/debugging.md](references/debugging.md).
4. **Unsure whether a feature exists**: check [references/implementation-status.md](references/implementation-status.md) and [references/builtins.md](references/builtins.md) before writing it. Never invent built-ins or library functions — check [references/stdlib.md](references/stdlib.md) or grep `stdlib/`.
5. **Changing the compiler**: Tier 5 in full, plus [references/pipeline.md](references/pipeline.md).

### Rule Application by Task

| Task | Primary categories |
|---|---|
| New function | `fn-`, `type-`, `match-`, `err-` |
| New data type | `data-`, `match-`, `gen-`, `trait-` |
| Printing / strings / floats | `fn-print-semantics`, `fn-types-drive-codegen`, `fn-string-equality`, `data-field-types`, `trait-derive-show` |
| Parsing / text | `data-views-and-slices`, `proj-idioms`, `syn-string-literals` |
| Lists and data literals | `syn-list-literals-and-quote`, `data-collections-persistent` |
| Error handling | `err-`, `contract-`, `fn-unlowered-forms` |
| Mutation / state | `own-`, `data-struct-immutable-rebind`, `data-collections-persistent` |
| Generic / reusable code | `gen-`, `trait-`, `closure-` |
| Macros | `macro-`, `syn-list-literals-and-quote` |
| Concurrency | `actor-`, `det-` |
| Calling C | `ffi-extern-required`, `ffi-`, `pkg-native-dependencies`, `bits-` |
| Crypto / key material | `secret-`, `crypto-`, `bits-` |
| Tests | `test-` |
| Multi-file programs, packages | `pkg-`, `test-program-library-tests-split` |
| Performance / memory | `own-heap-never-freed`, `own-with-region`, `own-stack-promotion`, `data-views-and-slices`, `icnf-reuse-pass`, `cg-native-backend-mir` |
| Code review | [pitfalls](references/pitfalls.md), all **[CRITICAL]** rules |
| Compiler change | `boot-`, `pass-`, `icnf-`, `cg-`, `pass-evaluation-order-sets`, `det-` |
| Boot failure | `boot-failure-modes`, `tool-debugging-the-pipeline`, [debugging](references/debugging.md) |

### Where the Old Constraint List Went

The previous `SKILL.md` §2 "bootstrap constraint list" (cited by the book and `PROGRESS.md`) now lives in [boot-lifted-constraints](rules/boot-lifted-constraints.md), which maps each old number to its rule.

## Keeping This Skill Current

A stale skill produces wrong code with high confidence. When a gap is fixed (a form gets lowered, a check starts firing, a bug is closed), update the rule, its row in [implementation-status](references/implementation-status.md) and [pitfalls](references/pitfalls.md), and regenerate the index above from the rules' `>` lines. Sources: `book/src/`, `zyl_specification.txt`, `PROGRESS.md`, `docs/`, and the compiler source (authoritative).
