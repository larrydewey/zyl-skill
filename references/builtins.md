# Built-in Forms and Operations

Authority: `dispatch-special` in `stdlib/compiler/expr_inner.zyl`, `ic-op-of` in `icnf.zyl`. **NL** = parsed but not lowered (does nothing / evaluates to 0). **lib** = ordinary library function, not a form.

## Definitions

| Form | Notes |
|---|---|
| `(defn name (param...) body...)` | param = `name` or `(name Type)`; no return type; wrap multi-form bodies in `begin` |
| `defun` | not reliably recognized — use `defn` |
| `(def name v)` | top level: **not readable** in compiled code; REPL only |
| `(deftype Name (Variant FieldType...) ...)` | nullary `(V)` or `V`; unknown uppercase field types are type params |
| `(defstruct Name f (f) (f Type)...)` | constructor `make-Name`; also `(Name ...)`; field types dropped |
| `defstruct+` | same as `defstruct`; `(:derive [...])` no-op |
| `(trait Name (method params...) ...)` | documentation + orphan-rule locality only |
| `(impl Trait Type (defn m (self ...) ...) ...)` | call `(Trait.m recv ...)` |
| `(derive Type Trait...)` | no-op |
| `(alias Name Type)` | no-op |
| `(defmacro name (params) template)`, `macro` | top level only; one body form |
| `(use path ...)`, `(pub <def>)`, `(feature-gate f <def>)`, `(module n)` (ignored), `(export n)` (dropped) | |

## Bindings and control

| Form | Notes |
|---|---|
| `(let name v body...)` / `(let (name v) body)` | one binding; paren form keeps **one** body form |
| `(let-mut name v body...)`, `(set! name v)` | `set!` only on `let-mut`/`for` vars |
| `(fn (p...) body)`, `(lambda ...)` | capture by value |
| `(if c t e)` | missing else → 0 |
| `(cond (t b) ... (else b))` | no match → 0 |
| `(while c body...)` | for effect |
| `(for ((i 0) (j 1)) c body)`, `(for (i 0) c body)`, `(for () c body)` | no step: body must `set!` |
| `(begin e...)` | last value; empty → 0 |
| `(match e arms...)` | constructor or literal arms, never mixed |
| `(try e (catch v h))` | catches `error` panics only |
| `(with-resource (n init) body)` | binds; **no release** |
| `and`, `or`, `not` | desugared to `if`; short-circuit; `(or 5 6)` → true |
| `(error "msg")` | lib (`allocator/allocator`); panics / unwinds to `try` |
| `when`, `unless` | lib (`core/core`); **eager** functions |
| `assert`, `unwrap` | **NL** |

## Arithmetic, comparison, bits

| Op | Notes |
|---|---|
| `+ - *` | n-ary fold left; only `(- x)` unary; `(+ x)` `(* x)` → 0 |
| `/` `%` | trunc toward zero; `%` sign of dividend; /0 → SIGFPE; Secret operand rejected |
| `= == != < > <= >=` | `=`≡`==`; strings by content only if kinds known; structs/ADTs shallow structural |
| `bit-and bit-or bit-xor` | n-ary |
| `bit-not` | exactly 1 arg |
| `shl shr ashr` | `shr` logical, `ashr` arithmetic; counts ≥ 64 defined (0 / sign fill) |

No overflow checks; bitwise ops not constant-folded.

## Strings

| Name | Notes |
|---|---|
| `(str-concat a b)` | inlined, fresh string; `(str-concat "" c-ptr)` copies a C string |
| `(str-length s)` | bytes; inlined |
| `(str-substring s start len)` | byte-indexed fresh copy; inlined |
| `(str-equal a b)` | content, 1/0; inlined |
| `str-eq`, `str-len`, `str-intern arena s`, `buf-append dst src` | lib (`allocator/allocator`) |

## Data

`(struct-get v "field")`, `(make-Name ...)`, constructor application. `make-struct`, `make-variant`: **NL**. `tuple`: undefined.

## I/O

| Form | Notes |
|---|---|
| `(print e...)` | each arg on its own line; returns 0; format by static kind |
| `print-int`, `print-float`, `print-string`, `print-bool` | lib (`core/core`), typed params |
| `(file-open path mode)` | fd; negative on failure; modes `"r"`, `"a"`, else write |
| `(file-read fd n)`, `(file-write fd text)`, `(file-close fd)` | |
| `read-line`, `exit`, `close` | **NL** |

## Actors and FFI

| Form | Notes |
|---|---|
| `(spawn entry)` | zero-arg, non-capturing; returns Int id |
| `(send a msg)` | queued then **discarded**; abort if `a` stopped |
| `(ffi-call "sym" args... timeout)` | last arg always dropped as timeout; not enforced |
| `(ffi-pin v)` / `(ffi-unpin p)` | pointer to a pin slot / read slot |
| runtime via `ffi-call` | `zyl_actor_send_closure a fn word`, `zyl_actor_wait_all`, `zyl_cstr_from_int arena n`, `zyl_now_ms`, `zyl_argc`, `zyl_arg_str`, `zyl_span_copy` |

## Bytes and atomics

`(byte n)`, `(bytebuf Region cap)`, `bytebuf-cap/len/ptr`, `(byteslice buf off len)`, `(byteslice-sub s off len)`, `(bytebuf-append dst slice)`, `(load-u8 :le buf off)`, `load-i8`, `(store-u8 :le buf off v)`, `store-i8`, `bytebuf-atomic-{load,store,add,sub,fetch-add,max,min,cas}`, `(align-check ptr n)`. Wider widths → `E_RESERVED_KEYWORD`.

## Contracts (evaluated/ignored)

`requires`, `ensures` (condition evaluated, discarded), `recover` (body only), `checkpoint` (identity), `contracts off FORM` (FORM). `invariant`: undefined function.

## Testing

`(test "name" body...)`, `(run-tests)`, `assert-equal`, `assert-true`, `assert-false` work. `assert-fail` always passes; `test-suite` drops its tests; `setup`, `teardown`, `test-property`, `test-compile`: not run.

## Types, regions, capabilities (annotation/argument names)

Types `Int Float Bool String Unit Byte List Option Result Vec Map`; regions `Stack Heap Global Circular Pin`; capabilities `Secret` (`TCap`/`TMut` are inferred). `declassify`, `ct-eq-bool`, `ct-eq-words-bool` drop `Secret`.
