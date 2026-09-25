# Migration: Rust, C/C++, Lisp → Zyl

## Rust

| Rust | Zyl |
|---|---|
| `let x = v;` / `let mut x` | `(let x v body)` / `(let-mut x v body)` + `(set! x v)` |
| `&T` / `&mut T` | TCap (every `let`, params) / TMut (`let-mut`) — inferred |
| borrow checker, lifetimes | name-based capability checks; no lifetimes; regions inferred per call (frame, caller's result region, or heap) |
| `Box<T>` | nothing to write; recursive fields are pointers |
| `Arc<Mutex<T>>` | actor-owned state; `atomic/atomic` or `bytebuf-atomic-*` for counters |
| `struct` with `p.x = 1` | `defstruct`, immutable fields; rebuild + rebind (a unique, dead old value is reused in place by the compiler) |
| `enum` + `match` | `deftype` + exhaustive `match`; no nested patterns (`E_NESTED_PATTERN`); guards only on literal arms |
| `4..=9 if v =>` | `((range 4 9) …)`; a guard after `range` is `E_ARITY_MISMATCH` — test in the body |
| `Option`/`Result`, `?` | same types; **no `?`**: nested `match` or `result-and-then` |
| `unwrap()` | `result-expect r "msg"` or `result-unwrap r default`; the `unwrap` form takes only an `Option` (`None` panics `unwrap on None`) |
| `panic!` / `catch_unwind` | `(error "msg")` / `(try e (catch m h))` |
| `()` | `Unit`, value `unit`; `print`, `set!`, `while` return it |
| `fn main()` | `(defn main () … 0)`: `main` returns an `Int`, the exit status |
| `fn f<T: Ord>(a: T)` | `(defn f (a) …)` — no type-parameter or bound syntax (`((T : Ord) a)` and `(a Ord)` are `E_MALFORMED_PARAMETER`); specialized per type at each call, a missing impl reported there |
| traits, `x.m()` | `(trait Tr (m (self (o Self)) Int))`, `(impl Tr Type …)`, `(Tr.m x)` or `(x.m)`; static dispatch only; no dyn, defaults, assoc types |
| `#[derive]` | `(derive T Show Debug Eq Ord Hash Clone)`, field types checked |
| closures `\|x\| x+n` | `(fn (x) (+ x n))`, capture by value, no mutation of captures |
| `macro_rules!` | `defmacro` templates, hygienic; `&rest xs` collects the remaining arguments and `,@xs` splices them; arguments are substituted, so one used twice runs twice |
| `vec![1, 2]` | `[1 2]` / `(list 1 2)` is a `List`; a `Vec` is built with `vec-create` + `vec-push` |
| `&s[a..b]`, `&v[a..b]` | `(view-slice s a (- b a))` (`text/view`, zero-copy `StrView`), `(slice-vec v a (- b a))` (`collections/slice`) |
| threads + channels | `spawn` + `send`/`(receive)`, reply to `(actor-self)`; `receive` is untyped |
| `extern "C"` | `(extern "sym" (Int String) Int)` + `(ffi-call "sym" args… timeout)`; word-sized types only, no `f64` |
| Cargo.toml / ranges | `zyl.pkg` / bare minimum versions (MVS) |
| Cargo.lock | `zyl.lock` (integrity record, commit it) |
| crates.io | git index, mandatory Ed25519, TOFU key pinning |
| features | additive `feature-gate` |
| build.rs | forbidden; declarative `(native …)` |
| `#[test]` | top-level `(test "name" …)` + `(run-tests)` |
| `println!("{}", x)` | `(print x)` (one line per arg); `str-concat` for text |
| `&str == &str` | `=` or `str-eq` (both compare content) |
| `as f64` | no casts; `(ffi-call "zyl_f_of_int" n 1000)` converts an Int; mixing is `E_TYPE_MISMATCH` |

## C / C++

| C | Zyl |
|---|---|
| `malloc`/`free` | automatic: per-call regions reclaim values that die in their call, escaping values live in the heap to exit; `with-region` for bounded arenas of typed values; `allocator/allocator` for raw memory |
| raw pointers | only via `ffi-pin` (`(Pin a)`), `bytebuf-ptr`, allocator (`Ptr`); no casts |
| `char *p = s + i` | `StrView` / `Cursor` (`text/view`): substrings and parsing without copies |
| mutable structs | immutable; rebind |
| headers | files are modules; `pub` marks exports; `(use m { a b })` |
| error codes / errno | `Result` |
| pthread + mutex | actors (isolated, no shared mutable state) |
| `#define MAX(a,b)` | `defmacro` on the AST; arguments still evaluated per use |
| `void f(...)` | `&rest` macro parameters; functions have fixed arity |
| Makefile | `zyl file.zyl` or `zyl build --locked`; reproducible builds |
| `int` overflow UB | wraps silently; `/0` SIGFPE |
| `if (n)` | `(if (!= n 0) …)`: conditions are `Bool` |
| `>>` on signed | `ashr` (arithmetic) vs `shr` (logical); counts ≥ 64 defined |
| `0xFFFF…` constants | write as negative Int if ≥ 2^63 |
| `double` to/from C | not possible: `extern` rejects `Float` |

## Lisp / Scheme / Clojure

| Lisp | Zyl |
|---|---|
| `'(1 2 3)`, `(list 1 2 3)` | the same: a typed `List`; `[1 2 3]` too |
| `'x`, symbols as data | none: a name inside quoted data is `E_MALFORMED_FORM` (no symbol type); keywords only inside specific forms |
| `` `(1 ,x ,@ys) `` | the same, as a list-building expression (all elements one type); no nested quasiquote |
| `(defmacro m (a &rest body) `(… ,@body))` | `(defmacro m (a &rest body) (… ,@body))`: the template is substituted directly, no quasiquote needed; exactly one template form |
| `#\| … \|#` | none; `;` only |
| `(lambda (x) …)` | `fn` or `lambda` |
| `define` at top level | `defn` for functions, `def` for constants |
| `let*`, named `let`, `letrec` | nested `let`; top-level helper `defn`s |
| `(let ((x 1) (y 2)) …)` | `E_MALFORMED_FORM` — nest lets |
| truthiness of non-nil | `Bool` only (`(if 0 …)` is `E_TYPE_MISMATCH`); a one-armed `if` is `Unit` |
| `car`/`cdr` | `Cons`/`Nil` ADT; `car`/`cdr` return `Option` |
| `eval` | none in compiled programs (REPL interprets entries) |
| dynamic typing | static Hindley–Milner inference, enforced: every type error is reported |
| heterogeneous lists | not possible; define an ADT for the element |
| GC | region/arena memory |
| `when`/`unless` macros | eager functions in the prelude with `Unit` bodies; define macros for laziness |

## Python / JS

No `null` (use `Option`), no truthiness (conditions are `Bool`), no mixing of ints and floats, errors are values (`Result`), immutable by default, compiled to a native binary, one-line output via `str-concat`.
