# Migration: Rust, C/C++, Lisp → Zyl

## Rust

| Rust | Zyl |
|---|---|
| `let x = v;` / `let mut x` | `(let x v body)` / `(let-mut x v body)` + `(set! x v)` |
| `&T` / `&mut T` | TCap (every `let`, params) / TMut (`let-mut`) — inferred |
| borrow checker, lifetimes | name-based capability checks; no lifetimes; regions inferred (heap by default) |
| `Box<T>` | nothing to write; recursive fields are pointers |
| `Arc<Mutex<T>>` | actor-owned state; `atomic/atomic` or `bytebuf-atomic-*` for counters |
| `struct` with `p.x = 1` | `defstruct`, immutable fields; rebuild + rebind |
| `enum` + `match` | `deftype` + exhaustive `match`; no nested patterns; guards only on literal arms |
| `4..=9 if v =>` | `((range 4 9) …)`; guard after range is broken — use literals or test in body |
| `Option`/`Result`, `?` | same types; **no `?`**: nested `match` or `result-and-then` |
| `unwrap()` | `result-expect r "msg"` or `result-unwrap r default` (**`unwrap` form → 0**) |
| `panic!` / `catch_unwind` | `(error "msg")` / `(try e (catch m h))` |
| `fn f<T: Ord>(a: T)` | `(defn f (a) …)` — no type-parameter syntax |
| traits, `x.m()` | `(trait …)`, `(impl T Type …)`, `(T.m x)`; no dyn, defaults, assoc types |
| `#[derive]` | `(derive …)` accepted, no effect |
| closures `\|x\| x+n` | `(fn (x) (+ x n))`, capture by value, no mutation of captures |
| `macro_rules!` | `defmacro` templates, hygienic, no quasiquote, plain params |
| threads + channels | `spawn` + closure messages via `zyl_actor_send_closure`; `send` is discarded |
| `extern "C"` | `(ffi-call "sym" args… timeout)`; int64/pointers only |
| Cargo.toml / ranges | `zyl.pkg` / bare minimum versions (MVS) |
| Cargo.lock | `zyl.lock` (integrity record, commit it) |
| crates.io | git index, mandatory Ed25519, TOFU key pinning |
| features | additive `feature-gate` |
| build.rs | forbidden; declarative `(native …)` |
| `#[test]` | top-level `(test "name" …)` + `(run-tests)` |
| `println!("{}", x)` | `(print x)` (one line per arg); `str-concat` for text |
| `&str == &str` | `str-eq` |
| `as f64` | no conversions |

## C / C++

| C | Zyl |
|---|---|
| `malloc`/`free` | automatic heap arena (never freed) or `allocator/allocator` arenas |
| raw pointers | only via `ffi-pin`, `bytebuf-ptr`, allocator (as `Int`) |
| mutable structs | immutable; rebind |
| headers | files are modules; `pub` marks exports; `(use m { a b })` |
| error codes / errno | `Result` |
| pthread + mutex | actors (isolated, no shared mutable state) |
| `#define MAX(a,b)` | `defmacro` on the AST; arguments still evaluated per use |
| Makefile | `zyl file.zyl` or `zyl build --locked`; reproducible builds |
| `int` overflow UB | wraps silently; `/0` SIGFPE |
| `>>` on signed | `ashr` (arithmetic) vs `shr` (logical); counts ≥ 64 defined |
| `0xFFFF…` constants | write as negative Int if ≥ 2^63 |

## Lisp / Scheme / Clojure

| Lisp | Zyl |
|---|---|
| `'x`, `` `(… ,x) `` | none — and the characters truncate the file |
| `#| … |#` | none; `;` only |
| `(lambda (x) …)` | `fn` or `lambda` |
| `define` at top level | `defn`; top-level `def` unreadable in compiled code |
| `let*`, named `let`, `letrec` | nested `let`; top-level helper `defn`s |
| `(let ((x 1) (y 2)) …)` | **compiles wrong** — nest lets |
| truthiness of non-nil | Bool only; one-armed `if` → 0 |
| `car`/`cdr` | `Cons`/`Nil` ADT; `car`/`cdr` return `Option` |
| `eval` | none in compiled programs (REPL interprets entries) |
| dynamic typing | static inference (not enforced) |
| GC | region/arena memory |
| `when`/`unless` macros | eager functions in the prelude; define macros for laziness |
| symbols as data, keywords as values | keywords only inside specific forms |

## Python / JS

No `null` (use `Option`), errors are values (`Result`), immutable by default, compiled to a native binary, one-line output via `str-concat`.
