# Built-in Forms and Operations

Authority: `dispatch-special` in `stdlib/compiler/expr_inner.zyl`, `ic-op-of` in `icnf.zyl`. **NL** = parsed but not lowered (does nothing / evaluates to 0). **lib** = ordinary library function, not a form.

## Definitions

| Form | Notes |
|---|---|
| `(defn name (param...) body...)` | param = `name` or `(name Type)`; no return type; wrap multi-form bodies in `begin` |
| `defun` | not reliably recognized — use `defn` |
| `(def name v)` | top level: **not readable** in compiled code; REPL only |
| `(deftype Name (Variant FieldType...) ...)` | nullary `(V)` or `V`; unknown uppercase field types are type params |
| `(defstruct Name f (f) (f Type)...)` | constructor `make-Name`; also `(Name ...)`; field types checked at constructor calls (definite clashes), then dropped |
| `defstruct+` | same as `defstruct`; `(:derive [...])` not parsed |
| `(trait Name (method params...) ...)` | documentation + orphan-rule locality only |
| `(impl Trait Type (defn m (self ...) ...) ...)` | call `(Trait.m recv ...)` |
| `(derive Type Trait...)` / `(derive Type [Trait...])` | `Show` generates an impl (`Secret` fields → `<secret>`, `impl-not Show` fields → `<hidden>`); other traits no-op |
| `(impl-not Trait Target)` | top-level; `Target` a type or a trait (all implementors); any impl/derive of the pair, or an impl of `Trait` whose result derives from a protected value, is `E_IMPL_FORBIDDEN`; for `Show` the target gets a compiler-made `<hidden>` Show |
| prelude `(trait Secret (wipe (self) Int))` | `(impl Secret T ...)` makes `T` key material: constructors yield secrets, params/fields of `T` are secret, prints `<secret>`, `(k.wipe)` erases; prelude `(impl-not Show Secret)` |
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
| `assert`, `unwrap` | lowered; panic on failure without the message ([err-no-assert-unwrap](../rules/err-no-assert-unwrap.md)) |

## Arithmetic, comparison, bits

| Op | Notes |
|---|---|
| `+ - *` | n-ary fold left; only `(- x)` unary; `(+ x)` `(* x)` → 0 |
| `/` `%` | trunc toward zero; `%` sign of dividend; /0 → SIGFPE; Secret operand rejected |
| `= == != < > <= >=` | `=`≡`==`; Strings by content (`<` byte order) when their type is known; structs/ADTs: `==`/`!=` deep by content (generated `T.==`; shallow words for Secret-field or unknown types), `<` etc. raw field words |
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
| `(print e...)` | each arg on its own line; returns 0; format by inferred type; a type with a `Show` impl prints `(Show.show e)` |
| `print-int`, `print-float`, `print-string`, `print-bool` | lib (`core/core`), typed params |
| `(file-open path mode)` | fd; negative on failure; modes `"r"`, `"a"`, else write |
| `(file-read fd n)`, `(file-write fd text)`, `(file-close fd)` | |
| `read-line`, `exit`, `close` | **NL** |

## Actors and FFI

| Form | Notes |
|---|---|
| `(spawn entry)` | zero-param fn or named fn; immutable captures OK, a parameter gets 0; returns Int id |
| `(send a msg)` | async, FIFO per sender; one word (Int or immutable heap pointer); no-op if `a` not live; dropped if `a` never receives |
| `(receive)` | next data message of the running actor (or `main`); blocks; queued closure messages ahead run first |
| `(actor-self)` | running actor's id; on `main`, opens a mailbox so actors can reply |
| `(ffi-call "sym" args... timeout)` | last arg always dropped as timeout; not enforced |
| `(ffi-pin v)` / `(ffi-unpin p)` | pointer to a pin slot / read slot |
| runtime via `ffi-call` | `zyl_actor_send_closure a fn word`, `zyl_actor_wait_all`, `zyl_cstr_from_int arena n`, `zyl_now_ms`, `zyl_argc`, `zyl_arg_str`, `zyl_span_copy` |

## Bytes and atomics

`(byte n)`, `(bytebuf Region cap)`, `bytebuf-cap/len/ptr`, `(byteslice buf off len)`, `(byteslice-sub s off len)`, `(bytebuf-append dst slice)`, `(load-u8 :le buf off)`, `load-i8`, `(store-u8 :le buf off v)`, `store-i8`, `load-{u,i}{16,32,64}`, `store-{u,i}{16,32,64}` (same shape), `bytebuf-atomic-{load,store,add,sub,fetch-add,max,min,cas}`, `(align-check ptr n)`. Handles are typed `ByteBuf` / `ByteSlice`.

## Contracts

`(requires C)`, `(invariant C)` (checked where written), `(ensures C)` (leading `defn` body form; checked after the body, value bound to `result`) — failure panics `E_CONTRACT_VIOLATION: precondition of f failed: C` (`postcondition`, `invariant`), catchable by `try`. `(recover BODY ((T) fallback) ...)` = `(try BODY (catch _ fallback))` with the first arm. `(checkpoint E)` = `E`. `(contracts off FORM)` strips contracts inside `FORM` (lexically); bare top-level `(contracts off)` strips the next form. See [contract-not-enforced](../rules/contract-not-enforced.md).

## Testing

`(test "name" body...)`, `(run-tests)`, `assert-equal`, `assert-true`, `assert-false` work. `assert-fail` always passes; `test-suite` drops its tests; `setup`, `teardown`, `test-property`, `test-compile`: not run.

## Types, regions, capabilities (annotation/argument names)

Types `Int Float Bool String Unit Byte List Option Result Vec Map`; regions `Stack Heap Global Circular Pin`; capabilities `Secret` — a parameter or field annotation, or a trait (`TCap`/`TMut` are inferred). `declassify`, `ct-eq-bool`, `ct-eq-words-bool` drop `Secret`.
