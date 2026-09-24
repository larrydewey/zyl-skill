# Specification vs Implementation

The spec (`zyl_specification.txt` v5.0) is normative for the language; the source code is authoritative for what a program does today. Write code against the right-hand column.

| Area | Spec | Today |
|---|---|---|
| Type checking | HM with capability/trait constraints, errors rejected | inferred, **not enforced**; drives print formats, float ops, pinnability |
| Generic functions | `((T : Bound) x)` groups, monomorphized | groups do not parse; unannotated params are polymorphic, one shared body |
| Generic ADTs | yes | yes (same-type constraint not enforced); no generic structs |
| Traits | static resolution, bounds, coherence | qualified calls, runtime tag dispatch (correct for structs / single impl); orphan rule enforced; C1 fails in assembler; no bounds, defaults, dyn, assoc types |
| derive / alias | generate impls / transparent alias | no-ops |
| Tuples, quoted data, `defun`, named let, `let*` | yes | no |
| Top-level `def` / Global region | constants | unreadable in compiled code; REPL only |
| Regions | Stack/Heap/Global/Circular/Pin, R1–R8 | one stack-promotion shape; heap bump arena never freed; pin arena; no Global/Circular; `E_REGION_ESCAPE` never raised |
| Capabilities | TCap/TMut/TAtomic/TBox/TPin inferred | TCap/TMut by name (`let`/`let-mut`); TPin from `ffi-pin`; no TAtomic/TBox constructs |
| Closures | capture inference, region-assigned | capture by value; set! of captures rejected; recursive lambdas unsupported; spawn captures crash |
| Match | exhaustive constructor patterns | exhaustive (constructor-name based); literal/OR/range/guard extension; no nested patterns; duplicate arms unreported |
| Numerics | checked overflow, `E_DIVISION_BY_ZERO` | wrapping; SIGFPE (interpreter reports it); oversized literals → 0 |
| Errors | `error` returns `(Err msg)`; `assert`, `unwrap` | `error` panics/unwinds to `try`; `assert`/`unwrap` panic without their message; try hang bug on 2/4-arg fns |
| Contracts | profiles, injection phase 10 | parsed; conditions evaluated and ignored; `invariant` undefined |
| Actors | receive, deterministic FIFO, Send checks | spawn + closure messages via FFI; `send` discarded; no receive; OS-scheduled (non-deterministic output); `let-mut`/Secret syntactic checks |
| FFI | Pin + pinnable + enforced timeout | direct SysV call, ints/pointers only, no floats; timeout dropped; pin needed only for Secret; no callbacks |
| Bytes | 8/16/32/64-bit, region rules | 8-bit only (others `E_RESERVED_KEYWORD`); region rules unenforced |
| Secrets | (implementation-defined) | syntactic taint checker; annotations stop at `math/secret/secret`; manual zeroize |
| Macros | hygienic, innermost-first, terminating | all implemented; plain-identifier params; no quasiquote |
| Testing | suites, fixtures, properties, compile tests | flat `test` + `run-tests` + 3 assertions; rest parsed/ignored |
| Stack safety | TCO or heap frames | no TCO; 64 GiB-reserved stack for `main`; actors ~8 MB |
| ICNF | SSA with region annotations | untyped structured tree; one region rewrite |
| Optimization | safe only | int constant folding + dead-branch elimination |
| Pipeline | 11 phases | see [pipeline.md](pipeline.md); region inference last; contract injection not wired |
| Packages (§31) | full | implemented; index URL placeholder; git deps not fetched; only root native block built; vendor unused; capability check skips `main`/tests |
| Hash finalization | ICNF hash etc. | `.buildinfo` with compiler, graph, asm hashes (asm instead of ICNF); graph hash not mixed into binary |
| Determinism | total | holds for single-threaded, FFI-free programs; assembly and binaries reproducible |
| Reserved keywords | `E_RESERVED_KEYWORD` for all | only byte-width names |
