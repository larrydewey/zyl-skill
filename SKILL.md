---
name: zyl
description: >
  Expert Zyl knowledge for writing, reviewing and debugging Zyl code and the
  self-hosted Zyl compiler. 172 rules in 26 categories across five tiers
  (foundations, language model, systems, engineering, compiler internals),
  prioritized by impact, plus reference tables for error codes, built-ins,
  the standard library, the pipeline and spec-vs-implementation status.
  Use when editing any *.zyl file or zyl.pkg manifest, answering questions
  about Zyl syntax or semantics, reviewing Zyl code, working in
  stdlib/compiler or selfhost/, or diagnosing boot/fixed-point failures.
triggers:
  - .zyl files, zyl.pkg, zyl.lock
  - zyl, zyl-self, zyl-lsp, zyl repl, zyl eval
  - selfhost, stage1, stage2, stage3, boot.sh, fixed point, reseed
  - stdlib/compiler, icnf, codegen, ic-, cg-, mr-, sb-
metadata:
  version: "2.0.0"
  sources:
    - book/src (The Zyl Programming Language, Parts I-V and Appendices A-F)
    - zyl_specification.txt v5.0
    - stdlib/compiler, selfhost, runtime (implementation authority)
---

# Zyl Expert Guide

Zyl is a deterministic Lisp systems language: S-expression syntax, Hindley–Milner inference with capability types (TCap/TMut), region-based memory, actors, a custom IR (ICNF), and x86_64 native code. The compiler is written in Zyl and reproduces itself byte for byte (`./boot.sh`). Strict left-to-right evaluation; same input, same output.

**The spec is the design; the compiler is the truth.** Much of the spec is only partly implemented, and many gaps fail *silently* (compile fine, compute wrong). Every rule here states current behavior. When a rule and the spec disagree, follow the rule; when a rule and the code disagree, the code wins — then fix the rule.

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

1. **The type checker rejects nothing.** Inference drives codegen but ill-typed programs compile. `(+ 1 "a")` and `(+ 1.5 2)` compute garbage. You are the type checker. → [type-inference-does-not-reject](rules/type-inference-does-not-reject.md)
2. **Codegen only knows a value is a String or Float from literals and annotations.** Unannotated params, fields, pattern binders, captures and polymorphic results print as integers and compare by address. Annotate `(s String)`/`(x Float)`, use `print-string`/`print-float`/`print-int`, compare strings with `str-eq`. → [fn-annotate-string-float-params](rules/fn-annotate-string-float-params.md)
3. **Stray characters end the file silently.** No `'` `` ` `` `,` `@` `#` outside strings/comments — no quote, quasiquote, block comments or commas in import lists. → [syn-no-stray-characters](rules/syn-no-stray-characters.md)
4. **Matches: spell constructors exactly, one level at a time, `_` last.** An unknown arm head is a catch-all; nested patterns are not tested; guards only on literal arms. → [match-misspelled-last-arm](rules/match-misspelled-last-arm.md)
5. **Several forms that parse do nothing:** `assert`, `unwrap` (→ 0), `read-line`, `exit`, contracts, `derive`, `alias`, `test-suite`, `assert-fail`, `with-resource` cleanup, top-level `def`. → [fn-unlowered-forms](rules/fn-unlowered-forms.md)
6. **Mutation is only `set!` on a `let-mut` name.** Params and `let` are immutable, struct fields are immutable (rebind the whole value), closures capture by value and cannot `set!` captures. → [own-let-mut-only-set](rules/own-let-mut-only-set.md)
7. **`let` binds one name; wrap multi-form bodies in `begin`.** Otherwise scopes leak and the parenthesized form drops forms. → [fn-begin-multi-form-bodies](rules/fn-begin-multi-form-bodies.md)
8. **`ffi-call` always drops its last argument as the timeout;** pass ints/pointers only (no floats); `ffi-pin` passes a pointer to a slot. → [ffi-timeout-always-last](rules/ffi-timeout-always-last.md)
9. **Actors: `send` is discarded, there is no `receive`, spawned closures must capture nothing, and `main` does not wait.** Use closure messages and `zyl_actor_wait_all`. → [actor-send-is-discarded](rules/actor-send-is-discarded.md)
10. **Compiler changes must reach a new fixed point:** `python3 selfhost/assemble.py && ./boot.sh --bootstrap-from-self && ./boot.sh`, commit the seed; keep every compiler file balanced on its own; introduce syntax in two steps. → [boot-fixed-point-workflow](rules/boot-fixed-point-workflow.md)

## Minimal Correct Program

```lisp
;; hello.zyl  --  zyl hello.zyl -o hello && ./hello
(deftype Shape (Circle Int) (Rect Int Int))

(defn area (s)
  (match s
    (Circle r (* 3 (* r r)))
    (Rect w h (* w h))))

(defn label ((name String) n)                ; String annotation: prints as text
  (begin
    (print-string (str-concat name ":"))
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
| 1 | 1 | Syntax & Lexical Structure | CRITICAL | `syn-` | 6 |
| 1 | 2 | Functions, Bindings & Control Flow | CRITICAL | `fn-` | 15 |
| 1 | 3 | Structs, ADTs & Collections | CRITICAL | `data-` | 11 |
| 1 | 4 | Pattern Matching | CRITICAL | `match-` | 8 |
| 1 | 5 | Error Handling | CRITICAL | `err-` | 5 |
| 1 | 6 | Contracts | HIGH | `contract-` | 1 |
| 2 | 7 | Ownership, Capabilities & Regions | HIGH | `own-` | 7 |
| 2 | 8 | Type System | CRITICAL | `type-` | 4 |
| 2 | 9 | Generics | HIGH | `gen-` | 5 |
| 2 | 10 | Traits | HIGH | `trait-` | 6 |
| 2 | 11 | Closures | HIGH | `closure-` | 4 |
| 2 | 12 | Macros | HIGH | `macro-` | 8 |
| 3 | 13 | FFI | CRITICAL | `ffi-` | 6 |
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
| 5 | 24 | ICNF | MEDIUM | `icnf-` | 5 |
| 5 | 25 | x86_64 Codegen | HIGH | `cg-` | 8 |
| 5 | 26 | Writing Compiler Passes | HIGH | `pass-` | 10 |
## Tier 1 — Foundations (every program)

### 1. Syntax & Lexical Structure (CRITICAL)

- [`syn-brackets-and-balance`](rules/syn-brackets-and-balance.md) - `()`, `[]` and `{}` all read as the same list; keep every opener matched, and know the balance check only catches net imbalance.
- [`syn-int-literal-range`](rules/syn-int-literal-range.md) **[CRITICAL]** - Write any constant at or above 2^63 as its negative two's-complement `Int`; never write an unsigned-sized literal.
- [`syn-keywords-and-symbols`](rules/syn-keywords-and-symbols.md) - Use `:keyword` tokens only where a form expects them; they are not values.
- [`syn-naming-conventions`](rules/syn-naming-conventions.md) - kebab-case functions and variables, PascalCase types and variants, `?` predicates, `_` for unused, two-space indent; treat special-form names as reserved.
- [`syn-no-stray-characters`](rules/syn-no-stray-characters.md) **[CRITICAL]** - Never write `'`, `` ` ``, `,`, `@`, `#`, `$`, `&`, `|`, `^`, `\` or non-ASCII bytes outside strings and comments.
- [`syn-string-literals`](rules/syn-string-literals.md) - Use only the supported escapes (`\n \t \r \0 \" \\ \e \xNN`); remember strings are NUL-terminated byte pointers.

### 2. Functions, Bindings & Control Flow (CRITICAL)

- [`fn-annotate-string-float-params`](rules/fn-annotate-string-float-params.md) **[CRITICAL]** - Annotate parameters that hold a `String` or `Float` — `(s String)`, `(x Float)` — whenever the function prints, compares or does arithmetic on them.
- [`fn-begin-multi-form-bodies`](rules/fn-begin-multi-form-bodies.md) **[CRITICAL]** - Wrap every body that has more than one form in `begin`.
- [`fn-conditionals`](rules/fn-conditionals.md) - Always give `if` an else branch and `cond` an `else` clause in value position; conditions must be real Bools.
- [`fn-for-has-no-step`](rules/fn-for-has-no-step.md) - `for` has no step clause: the body must `set!` the loop variable, or the loop never ends.
- [`fn-int-float-separation`](rules/fn-int-float-separation.md) **[CRITICAL]** - Never mix `Int` and `Float` in one operation; there is no conversion and no diagnostic.
- [`fn-integer-arith-unchecked`](rules/fn-integer-arith-unchecked.md) - Guard divisors and overflow yourself: compiled integer arithmetic wraps silently and division by zero kills the process with SIGFPE.
- [`fn-let-single-binding`](rules/fn-let-single-binding.md) **[CRITICAL]** - `let` binds exactly one name; nest `let`s for several, and prefer the bare `(let name value body)` spelling.
- [`fn-main-and-exit-status`](rules/fn-main-and-exit-status.md) - Every executable needs `(defn main () ...)` with no parameters; its value is the exit status, so end it with an explicit `0`.
- [`fn-no-named-let-or-early-return`](rules/fn-no-named-let-or-early-return.md) - There is no `return`, named `let`, `let*` or TCO: structure code as small recursive helpers with accumulators, or `while` loops.
- [`fn-no-return-type-slot`](rules/fn-no-return-type-slot.md) - Parameters are a bare name or `(name Type)`; there is no return-type annotation and no annotation on `let`.
- [`fn-no-toplevel-def`](rules/fn-no-toplevel-def.md) - Use a zero-argument `defn` for constants; a top-level `def` is not readable in compiled code.
- [`fn-print-semantics`](rules/fn-print-semantics.md) - `print` writes each argument on its own line and evaluates to 0; build one-line output with `str-concat`.
- [`fn-string-equality`](rules/fn-string-equality.md) **[CRITICAL]** - Compare strings with `str-eq` (or `str-equal`) unless both operands are literals, string built-in results, or `String`-annotated parameters.
- [`fn-underscore-discard`](rules/fn-underscore-discard.md) - Use `_` (or a `_`-prefixed name) for anything deliberately unused; never invent dummy names.
- [`fn-unlowered-forms`](rules/fn-unlowered-forms.md) **[CRITICAL]** - Do not use forms that parse but are not lowered: `assert`, `unwrap`, `read-line`, `exit`, `close`, `make-struct`, `make-variant`, `invariant`, and `with-resource` cleanup.

### 3. Structs, ADTs & Collections (CRITICAL)

- [`data-adt-declaration`](rules/data-adt-declaration.md) - Declare sum types with `(deftype Name (Variant FieldType...) ...)`; an unknown uppercase field type is a type parameter.
- [`data-collections-int-persistent`](rules/data-collections-int-persistent.md) **[CRITICAL]** - `Vec`, `Map` and `Set` hold `Int`s only and return an updated value: always rebind to the result and treat the old value as used up.
- [`data-equality-shallow`](rules/data-equality-shallow.md) - `==` and `<` on structs/ADTs compare one level deep; compare nested values and strings field by field yourself.
- [`data-field-kinds`](rules/data-field-kinds.md) **[CRITICAL]** - Treat every struct field and pattern-bound name as an untyped word: route `String` and `Float` values through typed functions before printing, comparing or doing arithmetic.
- [`data-no-redeclare-prelude`](rules/data-no-redeclare-prelude.md) - Never redeclare `Option`, `Result`, `List` or any prelude function name; pick another name.
- [`data-no-tuples-no-generic-structs`](rules/data-no-tuples-no-generic-structs.md) - Use a struct or a single-variant ADT where you want a tuple; use a generic ADT where you want a generic struct; don't rely on `alias`.
- [`data-reconstruct-field-order`](rules/data-reconstruct-field-order.md) **[CRITICAL]** - When rebuilding a struct or variant, pass every field in declaration order — double-check against the `defstruct`/`deftype`.
- [`data-struct-basics`](rules/data-struct-basics.md) - Declare with `defstruct`, build with `make-Name`, read with `(struct-get v "field")` — the field name is a string.
- [`data-struct-immutable-rebind`](rules/data-struct-immutable-rebind.md) - Struct fields never change: to "update" one, build a new struct and rebind a `let-mut` name to it.
- [`data-two-map-types`](rules/data-two-map-types.md) - Pick one map per program: `core/map` (string keys, `Option` results, persistent) or `collections/map` (Int keys/values, default value, arena-backed).
- [`data-unique-variant-names`](rules/data-unique-variant-names.md) **[CRITICAL]** - Give every variant a name unique across all types in the program, and declare each type name exactly once.

### 4. Pattern Matching (CRITICAL)

- [`match-arm-complex`](rules/match-arm-complex.md) - In a match arm, never combine a constant with two or more calls in one arithmetic expression; bind the calls with `let` first.
- [`match-arm-shape`](rules/match-arm-shape.md) - Write arms as `(Variant binder... body)` with exactly one binder (a name or `_`) per field; the grouped `((Variant binder...) body)` form means the same.
- [`match-catch-all-last`](rules/match-catch-all-last.md) - Put `_` last, exactly once; an arm after a catch-all is `E_UNREACHABLE_MATCH_ARM`.
- [`match-exhaustive-or-underscore`](rules/match-exhaustive-or-underscore.md) - Cover every variant, or end with a single `_` arm; a match that falls through evaluates to 0.
- [`match-guards-literal-arms-only`](rules/match-guards-literal-arms-only.md) **[CRITICAL]** - Use `(when cond)` guards only after plain literal alternatives; test constructor fields inside the arm body instead.
- [`match-literal-requires-underscore`](rules/match-literal-requires-underscore.md) - Literal, OR and range matches must end with `_`, bind nothing, and cannot be mixed with constructor arms.
- [`match-misspelled-last-arm`](rules/match-misspelled-last-arm.md) **[CRITICAL]** - Spell every constructor in a match arm exactly: an arm head that is not a known constructor is a catch-all that binds nothing.
- [`match-no-nested-patterns`](rules/match-no-nested-patterns.md) **[CRITICAL]** - Match one constructor level at a time; a field position holds only a name or `_`.

### 5. Error Handling (CRITICAL)

- [`err-helper-argument-order`](rules/err-helper-argument-order.md) - Option/Result helpers take the value first and the function second; `collections/collections` list helpers take the collection **last**; `-unwrap` helpers require a default.
- [`err-no-assert-unwrap`](rules/err-no-assert-unwrap.md) **[CRITICAL]** - Never use `assert` or `unwrap` in compiled code: `assert` checks nothing and `unwrap` evaluates to 0.
- [`err-result-for-expected-failures`](rules/err-result-for-expected-failures.md) - Return `Result` (`Ok`/`Err`) for failures a caller can handle, `Option` (`Some`/`None`) for absence; reserve `error` for unrecoverable conditions.
- [`err-try-catches-error-not-err`](rules/err-try-catches-error-not-err.md) **[CRITICAL]** - `try`/`catch` intercepts `error` panics only; an `(Err ...)` value passes straight through it.
- [`err-try-even-arity-hang`](rules/err-try-even-arity-hang.md) **[CRITICAL]** - Do not rely on catching an `error` raised inside a function called with an even number of arguments; prefer `Result` for failures you expect to handle.

### 6. Contracts (HIGH)

- [`contract-not-enforced`](rules/contract-not-enforced.md) - Treat contracts as documentation: `requires`/`ensures` conditions are evaluated and discarded, and `invariant`, result binding, profiles and rollback do not exist.


## Tier 2 — The Language Model

### 7. Ownership, Capabilities & Regions (HIGH)

- [`own-capability-kinds`](rules/own-capability-kinds.md) - Know the capability kinds and which ones source code can actually produce: `TCap` (let/params), `TMut` (let-mut/for), `TPin` (ffi-pin result), and the `Secret` annotation.
- [`own-heap-never-freed`](rules/own-heap-never-freed.md) - Expect heap values to live until process exit; in long-running or allocation-heavy code, manage memory with an explicit arena from `allocator/allocator`.
- [`own-let-copies-word`](rules/own-let-copies-word.md) - Know that every value is one 64-bit word and `let` copies the word: scalars become independent copies, pointers become aliases.
- [`own-let-mut-only-set`](rules/own-let-mut-only-set.md) - `set!` only a plain name bound by `let-mut` (or a `for` variable) in the current scope; everything else is immutable.
- [`own-no-closure-captured-mutation`](rules/own-no-closure-captured-mutation.md) **[CRITICAL]** - Never `set!` a captured variable inside a closure; have the closure return a new value and rebind it in the owning scope.
- [`own-regions-status`](rules/own-regions-status.md) - Write code against the regions that exist today: Stack (frames), one Heap arena, and Pin; Global and Circular are not implemented.
- [`own-stack-promotion`](rules/own-stack-promotion.md) - To keep a short-lived ADT value off the heap, bind it with `let` and only `match` on it or `print` it in the body.

### 8. Type System (CRITICAL)

- [`type-annotations-guide-codegen`](rules/type-annotations-guide-codegen.md) - Use parameter annotations to tell code generation what a value is, not as a safety net; they are optional and unchecked.
- [`type-inference-does-not-reject`](rules/type-inference-does-not-reject.md) **[CRITICAL]** - Be your own type checker: the compiler infers types but does not reject ill-typed programs.
- [`type-polymorphic-results-typed-printers`](rules/type-polymorphic-results-typed-printers.md) **[CRITICAL]** - Print or compare the result of a polymorphic (unannotated) function through a typed function: `print-int`, `print-string`, `print-float`, or a `(x Type)` parameter.
- [`type-value-representation`](rules/type-value-representation.md) - Reason about values as single 64-bit words with a fixed, documented layout.

### 9. Generics (HIGH)

- [`gen-generic-adts`](rules/gen-generic-adts.md) - Make data generic with ADTs whose field types are unknown uppercase names; don't expect the same-type constraint or generic structs.
- [`gen-monomorphization-naming`](rules/gen-monomorphization-naming.md) - Know that specialization names are canonical (sorted type names) and that the only per-type copies are impl methods named `Trait.method_Type`.
- [`gen-no-type-parameter-syntax`](rules/gen-no-type-parameter-syntax.md) **[CRITICAL]** - Never write the spec's type-parameter groups `((T) x)` or `((T : Ord) a b)`; leave parameters unannotated instead.
- [`gen-operators-not-overloaded`](rules/gen-operators-not-overloaded.md) - Pass comparison and combination functions explicitly in generic code; operators compare machine words.
- [`gen-unannotated-is-polymorphic`](rules/gen-unannotated-is-polymorphic.md) - Write generic functions by leaving parameters unannotated; each call site is inferred separately and all sites share one compiled body.

### 10. Traits (HIGH)

- [`trait-bind-before-struct-get`](rules/trait-bind-before-struct-get.md) - Bind a trait call's result with `let` before passing it to `struct-get`.
- [`trait-coherence-and-orphans`](rules/trait-coherence-and-orphans.md) - Declare the trait (`(trait Name ...)`) in your package before implementing it for a type you don't own, and write each `(Trait, Type)` impl exactly once.
- [`trait-derive-noop`](rules/trait-derive-noop.md) - Don't rely on `derive` or `defstruct+ (:derive ...)`: both are accepted and generate nothing.
- [`trait-dispatch-structs-only`](rules/trait-dispatch-structs-only.md) **[CRITICAL]** - Implement multi-impl traits for **structs**; give a trait over an ADT or primitive type a single impl.
- [`trait-no-dyn-use-adt-wrapper`](rules/trait-no-dyn-use-adt-wrapper.md) - Model open "trait object" designs with an ADT wrapper, a list of structs, or function values; there is no `dyn`.
- [`trait-qualified-calls`](rules/trait-qualified-calls.md) - Call trait methods by qualified name, `(Trait.method receiver args...)`, with the receiver first.

### 11. Closures (HIGH)

- [`closure-capture-by-value`](rules/closure-capture-by-value.md) - Expect a closure to see each captured variable's value at the moment the closure was created.
- [`closure-captured-kinds-lost`](rules/closure-captured-kinds-lost.md) **[CRITICAL]** - Don't `print`, `=`-compare or do float arithmetic directly on captured variables or on results of calls through function values; pass them to typed functions.
- [`closure-explicit-fn-syntax`](rules/closure-explicit-fn-syntax.md) - Write anonymous functions as `(fn (params) body)` or `(lambda (params) body)`; there is no shorthand.
- [`closure-no-recursive-lambda`](rules/closure-no-recursive-lambda.md) - Write recursive helpers as top-level `defn`s; a lambda cannot refer to itself.

### 12. Macros (HIGH)

- [`macro-args-spliced`](rules/macro-args-spliced.md) - Use each parameter once in a macro body, or bind it with `let` inside the body so the argument is evaluated exactly once.
- [`macro-hygiene`](rules/macro-hygiene.md) - Pass everything a macro needs from the call site as an argument; the body's binders are renamed and it cannot see the caller's locals.
- [`macro-name-positions`](rules/macro-name-positions.md) - Pass an identifier when a macro parameter lands in a name position (binder, `set!` target, `defn`/`deftype`/`impl` name).
- [`macro-no-recursion`](rules/macro-no-recursion.md) - Never write a macro that can expand into a call of itself, even behind an `if`; put recursion in functions.
- [`macro-prefer-functions`](rules/macro-prefer-functions.md) - Use a macro only when you need call-by-name evaluation (skip or repeat an argument) or new binding syntax; otherwise write a function.
- [`macro-shadows-functions`](rules/macro-shadows-functions.md) - Remember that a macro takes over every call to a same-named imported function, program-wide.
- [`macro-template-no-quasiquote`](rules/macro-template-no-quasiquote.md) **[CRITICAL]** - Write the macro body as the literal expansion: parameters are replaced by argument source; there is no quasiquote, and only one body form is kept.
- [`macro-top-level-and-registration`](rules/macro-top-level-and-registration.md) - Define macros only at top level, once per name; they may be used before their definition.


## Tier 3 — Systems Programming

### 13. FFI (CRITICAL)

- [`ffi-int64-only-no-floats`](rules/ffi-int64-only-no-floats.md) **[CRITICAL]** - Declare C functions called from Zyl with `int64_t` and pointer parameters and results only; never pass a `Float` to a C `double`.
- [`ffi-linking`](rules/ffi-linking.md) - Link your own C either with a package `native` block (preferred) or by emitting assembly and running `cc` yourself; the single-file CLI takes no extra objects or libraries.
- [`ffi-memory-across-boundary`](rules/ffi-memory-across-boundary.md) - Copy C strings into Zyl strings with `(str-concat "" ptr)` before freeing them; give C writable buffers from `alloc-malloc` or an arena.
- [`ffi-pin-passes-pointer`](rules/ffi-pin-passes-pointer.md) **[CRITICAL]** - Use `ffi-pin` only when C expects a **pointer to** a value (out-parameters); pass Ints and Strings directly.
- [`ffi-timeout-always-last`](rules/ffi-timeout-always-last.md) **[CRITICAL]** - Always end `ffi-call` with a timeout literal in milliseconds: `(ffi-call "sym" args... 1000)`.
- [`ffi-wrap-each-call`](rules/ffi-wrap-each-call.md) - Wrap every foreign function in one small, named Zyl function so the timeout, pinning and pointer conversion live in one place.

### 14. Actors (CRITICAL)

- [`actor-always-wait`](rules/actor-always-wait.md) **[CRITICAL]** - Wait for every actor before `main` returns: `actor-wait` per actor, or end with `(ffi-call "zyl_actor_wait_all" 1000)`.
- [`actor-closure-messages`](rules/actor-closure-messages.md) - Deliver work to a running actor with `(ffi-call "zyl_actor_send_closure" actor handler word 1000)`, where `handler` is a named one-parameter function.
- [`actor-limits`](rules/actor-limits.md) - Design within the runtime's limits: at most 1024 actors per process (ids never reused), ~8 MB actor stacks, unbounded mailboxes, and a panic in any actor kills the whole process.
- [`actor-no-let-mut-crossing`](rules/actor-no-let-mut-crossing.md) - Never reference a `let-mut` variable in a `send` message or spawned closure; snapshot it with `let` first.
- [`actor-output-nondeterministic`](rules/actor-output-nondeterministic.md) - Let exactly one actor (usually `main`) produce ordered output, or collect results and print after `zyl_actor_wait_all`.
- [`actor-send-is-discarded`](rules/actor-send-is-discarded.md) **[CRITICAL]** - Don't design around `send`/`receive`: data messages are queued and discarded, and there is no `receive`. Use closure messages to deliver work.
- [`actor-spawn-captures-nothing`](rules/actor-spawn-captures-nothing.md) **[CRITICAL]** - Pass `spawn` a named zero-argument function or a `fn` that captures nothing and takes no parameters.

### 15. Bits, Bytes & Buffers (HIGH)

- [`bits-atomics-aligned`](rules/bits-atomics-aligned.md) - Use `bytebuf-atomic-*` only at 8-aligned offsets that fit in the buffer, and know which ones return old vs new values.
- [`bits-bounds-fail-closed`](rules/bits-bounds-fail-closed.md) - Check the results of byte loads, stores and appends: out-of-range accesses do nothing and return 0 instead of failing.
- [`bits-bytebuf-basics`](rules/bits-bytebuf-basics.md) - Allocate packed bytes with `(bytebuf Region N)` using literal region and capacity; keep buffer handles in their own bindings.
- [`bits-defined-shift-counts`](rules/bits-defined-shift-counts.md) - Rely on Zyl's defined shift semantics (logical shifts by ≥64 give 0; `ashr` saturates), not on x86's mod-64 masking.
- [`bits-only-8bit-widths`](rules/bits-only-8bit-widths.md) - Build 16/32/64-bit values from 8-bit loads and shifts (or `math/bits` packing helpers); the wider load/store names are reserved and rejected.
- [`bits-regions-unenforced-use-pin`](rules/bits-regions-unenforced-use-pin.md) - Write `Pin` for any buffer whose address you take or share, even though region rules are not enforced yet.
- [`bits-shr-vs-ashr`](rules/bits-shr-vs-ashr.md) **[CRITICAL]** - Use `shr` (logical, zero fill) for bit patterns and `ashr` (arithmetic, sign fill) for signed numbers.

### 16. Secrets & Constant-Time (CRITICAL)

- [`secret-annotate-params`](rules/secret-annotate-params.md) - Mark key material with a `Secret` parameter annotation — `(k Secret)` or `(k (Secret Int))` — at every function that handles it.
- [`secret-branchless-masks`](rules/secret-branchless-masks.md) - Make secret-dependent decisions with masks from `math/secret/secret` (`ct-select`, `ct-eq`, `ct-is-zero`, `ct-mask`), never with `if`.
- [`secret-ct-eq-words-for-bytes`](rules/secret-ct-eq-words-for-bytes.md) **[CRITICAL]** - Compare MACs, tags, hashes and passwords with `ct-eq-words` / `ct-eq-words-bool`, never with `=` or a loop that stops at the first difference.
- [`secret-declassify-explicitly`](rules/secret-declassify-explicitly.md) - Make a secret-derived value public only through `declassify`, `ct-eq-bool` or `ct-eq-words-bool`, with a comment saying why it is safe.
- [`secret-five-prohibitions`](rules/secret-five-prohibitions.md) **[CRITICAL]** - Never branch on, index with, divide by, print, or send a `Secret`, and pass it to C only through `ffi-pin`.
- [`secret-unannotated-helpers-launder`](rules/secret-unannotated-helpers-launder.md) **[CRITICAL]** - Annotate every helper a secret flows through; an unannotated helper silently launders taint.
- [`secret-zeroize`](rules/secret-zeroize.md) - Erase key material explicitly with `(zeroize base n)` or `(zeroize-bytes base n)` when you are done with it.

### 17. Cryptography Library (MEDIUM)

- [`crypto-choose-primitives`](rules/crypto-choose-primitives.md) - Default to ChaCha20-Poly1305 for AEAD, X25519 + HKDF for key agreement, Ed25519 for signatures, `sysrng` for key material; never use `chacharng` for keys.
- [`crypto-representations`](rules/crypto-representations.md) - In `stdlib/math`, byte strings are word arrays with one byte (0..255) per 8-byte slot, big numbers are 24-bit limbs least-significant first, and every entry point takes an arena first.


## Tier 4 — Engineering & Tooling

### 18. Testing (HIGH)

- [`test-assert-equal-semantics`](rules/test-assert-equal-semantics.md) - Assert on scalars, strings and flat records; for nested data assert on individual fields or a computed count/sum.
- [`test-compiler-internals`](rules/test-compiler-internals.md) - Test compiler passes by `use`ing the compiler modules and asserting on their results, not only with black-box compile-fail files.
- [`test-program-library-tests-split`](rules/test-program-library-tests-split.md) - Structure a project as a library module (no `main`), a program file (`use` + `main`), and a test file (`use` + tests + `run-tests`).
- [`test-read-summary-line`](rules/test-read-summary-line.md) - Judge a test run by its output (`FAIL` lines and the `test result:` summary), not by the exit status.
- [`test-regression-runner`](rules/test-regression-runner.md) - Run `./run_regression_tests.sh --full --no-boot --filter <word>` for targeted checks; `--filter` narrows the selected mode, it does not select one.
- [`test-toplevel-forms-and-run-tests`](rules/test-toplevel-forms-and-run-tests.md) - Write tests as flat top-level `(test "name" body)` forms, end the file with `(run-tests)`, and do not define `main` in a test file.
- [`test-unimplemented-features`](rules/test-unimplemented-features.md) **[CRITICAL]** - Don't use `test-suite`, `setup`/`teardown`, `test-property`, `test-compile`, `assert-fail`, keyword options or the `run-tests-*` helpers: they compile and do nothing (or silently drop tests).

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
- [`pkg-stdlib-resolution`](rules/pkg-stdlib-resolution.md) - When a stdlib or compiler change "doesn't take effect", suspect a stale `~/.zyl`: set `ZYL_HOME=$PWD/build/boot` or re-run `./install.sh`.
- [`pkg-use-what-you-construct`](rules/pkg-use-what-you-construct.md) **[CRITICAL]** - Explicitly `use` every module whose types or constructors your module builds or matches on, even if they "happen to be visible".

### 20. Determinism (HIGH)

- [`det-left-to-right`](rules/det-left-to-right.md) - Rely on strict left-to-right evaluation everywhere, and never write code (or compiler passes) that reorders side effects.
- [`det-no-address-dependent-output`](rules/det-no-address-dependent-output.md) - Never let an address influence output: don't print pointers, don't compare strings or records by address, don't derive names or ordering from pointers.
- [`det-nondeterminism-sources`](rules/det-nondeterminism-sources.md) - Keep clock, environment, PID, kernel entropy and multi-actor output out of anything that must be reproducible — the compiler will not warn you.
- [`det-ordered-collections`](rules/det-ordered-collections.md) - Iterate only ordered structures (lists, association lists, insertion-ordered arrays); a hash table may be probed by key but never iterated.

### 21. Tooling (CLI, REPL, LSP) (MEDIUM)

- [`tool-cli-arguments`](rules/tool-cli-arguments.md) - Put the source file first: `zyl file.zyl [-o out] [--emit-asm]`; everything else is a subcommand, and unknown words become the output path.
- [`tool-debugging-the-pipeline`](rules/tool-debugging-the-pipeline.md) - Debug with `--emit-asm`, `ZYL_DEBUG_STAGES=1`, `zyl eval` differential runs, and small driver programs that `use` compiler modules.
- [`tool-eval-differential`](rules/tool-eval-differential.md) - Use `zyl eval` / the REPL for fast iteration, but confirm behavior with a compiled binary: the interpreter differs on actors, FFI, division by zero and speed.
- [`tool-lsp-and-editors`](rules/tool-lsp-and-editors.md) - Point any LSP client at `zyl-lsp` for the compiler's own diagnostics; expect one diagnostic at a time, no `W_` warnings, no capability check, and name-based (not scope-based) navigation.
- [`tool-repl`](rules/tool-repl.md) - Use the REPL (`zyl repl`) to explore expressions and definitions; remember `def` works there but not in compiled files, and each name can be defined once per session.

### 22. Project Idioms (MEDIUM)

- [`proj-buf-append-appends`](rules/proj-buf-append-appends.md) **[CRITICAL]** - Use `buf-append` only on a fresh zeroed buffer or one you intend to extend: it appends at `strlen(dst)`, it does not copy.
- [`proj-file-io`](rules/proj-file-io.md) - Use the built-in `file-open`/`file-read`/`file-write`/`file-close` forms; check the descriptor and read with an explicit maximum size.
- [`proj-idioms`](rules/proj-idioms.md) - Write Zyl in its native style: small pure functions, recursion with accumulators, ADTs plus exhaustive `match`, state threaded through return values, short `let` chains, I/O at the edges.


## Tier 5 — Compiler Contributors (`stdlib/compiler/`, `selfhost/`)

### 23. Bootstrap & Fixed Point (CRITICAL)

- [`boot-assemble-bundle`](rules/boot-assemble-bundle.md) - Know what `selfhost/assemble.py` does to the compiler source: it strips `use` lines and non-driver `main`s, drops duplicate `defn`s (first wins), and flattens everything into one namespace.
- [`boot-bind-calls-before-binop`](rules/boot-bind-calls-before-binop.md) - In compiler source, bind each call result with `let` before combining calls in one arithmetic or `str-concat` expression.
- [`boot-even-field-parity`](rules/boot-even-field-parity.md) - Give a record type that is passed as a constructor argument to another constructor call an even field count, until the workaround is re-tested.
- [`boot-failure-modes`](rules/boot-failure-modes.md) - Map a bootstrap failure to its cause before changing anything.
- [`boot-fixed-point-workflow`](rules/boot-fixed-point-workflow.md) - After editing `stdlib/compiler/*.zyl`, `selfhost/` or `runtime/actor_runtime.c`: re-bundle, reseed, verify, commit the seed.
- [`boot-lifted-constraints`](rules/boot-lifted-constraints.md) - Know which historical bootstrap constraints are lifted, so you neither follow dead rules blindly nor reintroduce the patterns they guarded against.
- [`boot-moderate-bodies`](rules/boot-moderate-bodies.md) - Keep compiler function bodies moderate and flat; prefer short `let` chains and helpers over deep nesting.
- [`boot-one-deftype-per-name`](rules/boot-one-deftype-per-name.md) **[CRITICAL]** - Define each type name exactly once across the whole bundle, and keep variant names unique.
- [`boot-parens-per-file`](rules/boot-parens-per-file.md) **[CRITICAL]** - Keep every top-level form, and every compiler-stdlib **file**, independently balanced; verify a hand-edited file by compiling it alone.
- [`boot-two-step-syntax`](rules/boot-two-step-syntax.md) - Introduce new syntax in two steps: teach the compiler to accept it and reseed, then start using it in the compiler's own source.

### 24. ICNF (MEDIUM)

- [`icnf-lowering-map`](rules/icnf-lowering-map.md) - Know what each source form lowers to, so you can predict codegen and write passes over ICNF.
- [`icnf-new-form-needs-case`](rules/icnf-new-form-needs-case.md) **[CRITICAL]** - Give every new special form its own case in `ic-expr-node`; lowering turns anything unrecognized into `(IConst 0)` silently.
- [`icnf-optimizer-scope`](rules/icnf-optimizer-scope.md) - Expect only integer constant folding (opcodes 0–10) and dead-branch elimination; never add an optimization that reorders or drops effects.
- [`icnf-regions-are-a-rewrite`](rules/icnf-regions-are-a-rewrite.md) - Region inference is one conservative ICNF rewrite (`IVariant` → `IStackVariant`); keep any extension conservative.
- [`icnf-tree-structure`](rules/icnf-tree-structure.md) - Treat ICNF as an untyped structured tree of `Icnf` nodes (not SSA), with three parameter representation kinds.

### 25. x86_64 Codegen (HIGH)

- [`cg-c-call-alignment`](rules/cg-c-call-alignment.md) **[CRITICAL]** - Route every C call through `cg-ext-call-aligned`, which forces 16-byte `rsp` alignment at the `call`.
- [`cg-call-arg-staging`](rules/cg-call-arg-staging.md) - Preserve `cg-call-args`' staging order: parity pad, each argument evaluated left to right into its own scratch slot, stack args copied, registers loaded, `call`, one cleanup `add`.
- [`cg-callee-saved-rbx-r12`](rules/cg-callee-saved-rbx-r12.md) **[CRITICAL]** - Use only `rbx` and `r12` among callee-saved registers in new codegen sequences, or extend `cg-save-callee-saved`/`cg-restore-callee-saved` first.
- [`cg-closure-call-protocol`](rules/cg-closure-call-protocol.md) - Calls through a local holding a function value go through `cg-call-indirect`, which passes one extra trailing argument: the closure env, or 0 for a plain code address.
- [`cg-emission-appends`](rules/cg-emission-appends.md) - Emit assembly by appending to the text buffer (`zyl_str_append` via `cg-emit*`), never by copying.
- [`cg-kind-of`](rules/cg-kind-of.md) - Remember codegen decides print format, float arithmetic and string/variant equality from a static `kind-of` (0 word, 1 String, 2 Float, 3 variant) with no return-type inference.
- [`cg-stack-machine`](rules/cg-stack-machine.md) - Model codegen as a stack machine over `rax` with `rbp`-relative slots and no register allocator.
- [`cg-symbols-and-entry`](rules/cg-symbols-and-entry.md) - User functions are labelled by mangled canonical keys (`zy_...`), the user entry is `_ZYL_main`, and every program shares one C `main` stub that runs it on a huge stack.

### 26. Writing Compiler Passes (HIGH)

- [`pass-adding-a-pass`](rules/pass-adding-a-pass.md) - Add a compiler pass by writing the module, calling it from `pipeline.zyl`, listing it in `assemble.py` before `pipeline.zyl`, testing it, and reseeding.
- [`pass-avoid-repeated-subtree-work`](rules/pass-avoid-repeated-subtree-work.md) - Visit each subtree once per pass; a pass that re-walks a subtree per visit goes exponential in nesting depth.
- [`pass-conservative-failure-direction`](rules/pass-conservative-failure-direction.md) - Make checks fail open (miss a diagnostic rather than reject valid code) and transformations fail safe (fall back to the always-correct path).
- [`pass-copy-spans`](rules/pass-copy-spans.md) - When a pass rebuilds an `Expr` node, copy the original's source span onto the replacement: `(ffi-call "zyl_span_copy" new-node old-node 1000)`.
- [`pass-diagnostics`](rules/pass-diagnostics.md) - Report new errors through `err-at` with a node and a catalogued code from `error_codes.zyl`, so they print located with a help line.
- [`pass-monomorphization-tables`](rules/pass-monomorphization-tables.md) - Treat `type-to-string` output as part of the fixed point: specialized symbol names are built from it.
- [`pass-no-stdout`](rules/pass-no-stdout.md) - Never write to stdout from a compiler pass; emit warnings on stderr.
- [`pass-state-threading`](rules/pass-state-threading.md) - Thread immutable state records through passes and return small wrapper ADTs when a function yields a value plus new state.
- [`pass-string-eq-in-compiler`](rules/pass-string-eq-in-compiler.md) **[CRITICAL]** - In compiler source, compare dynamically built strings (names, keys, type strings) with `str-eq`, not `=`.
- [`pass-total-structural-match`](rules/pass-total-structural-match.md) - In tree-rewriting passes, list every constructor explicitly; use `_` only in read-only checks that care about a few forms.

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
| Printing / strings / floats | `fn-annotate-*`, `fn-string-equality`, `data-field-kinds`, `type-polymorphic-*` |
| Error handling | `err-`, `contract-`, `fn-unlowered-forms` |
| Mutation / state | `own-`, `data-struct-immutable-rebind`, `data-collections-*` |
| Generic / reusable code | `gen-`, `trait-`, `closure-` |
| Macros | `macro-`, `syn-no-stray-characters` |
| Concurrency | `actor-`, `det-` |
| Calling C | `ffi-`, `pkg-native-dependencies`, `bits-` |
| Crypto / key material | `secret-`, `crypto-`, `bits-` |
| Tests | `test-` |
| Multi-file programs, packages | `pkg-`, `test-program-library-tests-split` |
| Performance / memory | `own-heap-never-freed`, `own-stack-promotion`, `closure-capture-by-value` |
| Code review | [pitfalls](references/pitfalls.md), all **[CRITICAL]** rules |
| Compiler change | `boot-`, `pass-`, `icnf-`, `cg-`, `det-` |
| Boot failure | `boot-failure-modes`, [debugging](references/debugging.md) |

### Where the Old Constraint List Went

The previous `SKILL.md` §2 "bootstrap constraint list" (cited by the book and `PROGRESS.md`) now lives in [boot-lifted-constraints](rules/boot-lifted-constraints.md), which maps each old number to its rule.

## Keeping This Skill Current

A stale skill produces wrong code with high confidence. When a gap is fixed (a form gets lowered, a check starts firing, a bug is closed), update the rule, its row in [implementation-status](references/implementation-status.md) and [pitfalls](references/pitfalls.md), and regenerate the index above from the rules' `>` lines. Sources: `book/src/`, `zyl_specification.txt`, `PROGRESS.md`, `docs/`, and the compiler source (authoritative).
