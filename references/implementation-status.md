# Specification vs Implementation

The spec (`zyl_specification.txt` v5.0) is normative for the language; the source code is authoritative for what a program does today. Write code against the right-hand column.

| Area | Spec | Today |
|---|---|---|
| Type checking | HM with capability/trait constraints, errors rejected | HM with let-polymorphism (`type_annotate.zyl`), **not enforced** (conflicts → unknown) except `E_TYPE_MISMATCH` for definite clashes with parameter/field annotations at calls; drives print formats, String/Float ops, trait resolution, per-type instances, generated ADT `==` |
| Generic functions | `((T : Bound) x)` groups, monomorphized | groups do not parse; unannotated params are polymorphic; shared body, plus `f~T` instances where the body depends on the type (≤32 each) |
| Generic ADTs | yes | yes (same-type constraint not enforced); no generic structs |
| Traits | static resolution, bounds, coherence | static resolution from inferred receiver types (runtime tag match only when unknown); `trait` signatures type calls; a call on a known type with no impl is located `E_TRAIT_NOT_FOUND`; prelude `Show Debug Eq Ord Hash Clone Secret`; `impl-not` (`E_IMPL_FORBIDDEN`, with flow rule, both located); orphan rule enforced; C1 duplicates are `E_DUPLICATE_IMPL`; no bounds, defaults, dyn, assoc types |
| derive / alias | generate impls / transparent alias | `derive` generates Show, Debug, Eq, Ord, Hash, Clone impls with field checks (`E_TRAIT_NOT_DERIVABLE`); prelude impls for primitives, List/Option/Result (Vec/Map only Show); `defstruct+ (:derive ...)` rewritten into a `derive`; `alias` no-op |
| Collections | `Vec<T>`, `Map<K,V>` | `(Vec T)` generic (`collections/vec`); `(Map String V)` (`core/map`, str-eq keys); `collections/map`/`set` Int-only |
| Tuples, quoted data, `defun`, named let, `let*` | yes | no |
| Top-level `def` / Global region | constants | immutable globals, initialized once in source order before `main` |
| Regions | Stack/Heap/Global/Circular/Pin, R1–R8 | one stack-promotion shape; heap bump arena never freed; pin arena; no Global/Circular; `E_REGION_ESCAPE` never raised |
| Capabilities | TCap/TMut/TAtomic/TBox/TPin inferred | TCap/TMut by name (`let`/`let-mut`); TPin from `ffi-pin`; no TAtomic/TBox constructs |
| Closures | capture inference, region-assigned | capture by value, captures keep their types; set! of captures rejected; recursive lambdas unsupported; spawn captures of immutable values work (`let-mut` captures `E_CAPABILITY_LEAK`) |
| Match | exhaustive constructor patterns | exhaustive (constructor-name based); literal/OR/range/guard extension; no nested patterns; duplicate arms unreported |
| Numerics | checked overflow, `E_DIVISION_BY_ZERO` | wrapping; SIGFPE (interpreter reports it); oversized literals → 0 |
| Errors | `error` returns `(Err msg)`; `assert`, `unwrap` | `error` panics/unwinds to `try` (any arity; the 2/4-arg hang is fixed); `assert`/`assert-true` show a string-literal message, else `assert failed`; `unwrap` panics `unwrap on None`; no `E_ASSERT_FAIL` |
| Contracts | profiles, injection phase 10 | `requires`/`ensures` (`result` bound)/`invariant` enforced, `E_CONTRACT_VIOLATION`, lowered in `expr_inner.zyl`; profiles strict/debug/warn/off/production via `--contracts=P` or `(contracts P [FORM])`; `recover` arms by error code, in order; `checkpoint` rolls back `let-mut` state (not byte buffers) and re-raises |
| Actors | receive, deterministic FIFO, Send checks | spawn; `send` + blocking `(receive)` + `(actor-self)` (main gets a mailbox), FIFO per sender, ADT messages; closure messages via FFI; spawn param always 0; not in the interpreter; OS-scheduled (non-deterministic output); `let-mut`/Secret syntactic checks |
| FFI | Pin + pinnable + enforced timeout | direct SysV call, ints/pointers only, no floats; timeout dropped; pin needed only for Secret; no callbacks |
| Bytes | 8/16/32/64-bit, region rules | 8/16/32/64-bit, le/be; `ByteBuf`/`ByteSlice` types; region rules unenforced |
| Secrets | (implementation-defined) | syntactic taint checker (impl bodies included); `Secret` fields and `Secret`-trait types, redacted as `<secret>` by `Show`; frames holding secrets zeroed on return (no tail calls); heap erasure manual (`zeroize`/`wipe`); annotations stop at `math/secret/secret`; a `let-mut` ever `set!` to a secret is secret for its scope; diagnostics located |
| Macros | hygienic, innermost-first, terminating | all implemented; plain-identifier params; no quasiquote |
| Testing | suites, fixtures, properties, compile tests | flat `test` + `run-tests` + 3 assertions; rest parsed/ignored |
| Stack safety | TCO or heap frames | tail calls are jumps, direct or through a function value, when stack args fit the caller's incoming area (not in `try`/`while` or frame-wiping secret functions); REPL interpreter TCO except String/Float results; otherwise 64 GiB-reserved stack for `main`; actors ~8 MB |
| ICNF | SSA with region annotations | untyped structured tree; one region rewrite |
| Optimization | safe only | int constant folding + dead-branch elimination |
| Pipeline | 11 phases | see [pipeline.md](pipeline.md); region inference last; contracts lowered in the front end (`expr_inner.zyl`), no separate injection phase |
| Packages (§31) | full | implemented; `ZYL_INDEX` selects an index (git URL or local path), `zyl publish --index DIR` adds to one, but the default hosted index does not exist yet; git deps cloned by revision; content-hash build cache for `zyl build`/`test`; only root native block built; vendor unused; capability check skips `main`/tests |
| Hash finalization | ICNF hash etc. | package builds only: `.buildinfo` with compiler, graph, native-object and ICNF hashes, the resolved graph, asm hash (informational) and final hash, embedded in the binary as `zyl_build_hash` |
| Determinism | total | holds for single-threaded, FFI-free programs; assembly and binaries reproducible |
| Reserved keywords | `E_RESERVED_KEYWORD` for all | none |
