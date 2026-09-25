# Built-in Forms and Operations

Authority: `dispatch-special` in `stdlib/compiler/expr_inner.zyl`, `ic-op-of` in `icnf.zyl`, the type rules in `type_annotate.zyl`. **NL** = parsed but not lowered (does nothing / evaluates to 0). **lib** = ordinary library function, not a form. Every form is type-checked and an ill-typed program does not compile: conditions are `Bool`, arithmetic never mixes `Int` and `Float`, statement forms (`print`, `set!`, `while`, `for`, `assert`, `send`, `if` without `else`) are `Unit`, whose value is written `unit`. A special form with the wrong shape is `E_MALFORMED_FORM`.

## Definitions

| Form | Notes |
|---|---|
| `(defn name (param...) body...)` | param = `name` or `(name Type)` (`(a : Int)` and `((T : Ord) a)` are `E_MALFORMED_PARAMETER`); no return type; several body forms are an implicit `begin`. `main` is `() -> Int` |
| `defun` | not recognized: its name stays undefined (`E_UNBOUND_VARIABLE` at the call) — use `defn` |
| `(def name v)` | top level: immutable global, evaluated once in source order before `main`/tests (`set!` on it is `E_MUT_CONFLICT`); in the REPL, later prompt definitions can use it |
| `(deftype Name (Variant FieldType...) ...)` | nullary `(V)` or `V`; unknown uppercase field types are type params; prelude constructor names (`Some None Ok Err Cons Nil`) are `E_DUPLICATE_VARIANT` |
| `(defstruct Name (f Type) (f)...)` | constructor `make-Name`; also `(Name ...)`; annotated fields are typed, an untyped field is an implicit type parameter of the struct |
| `defstruct+` | same as `defstruct`; a trailing `(:derive [Trait...])` is rewritten into `(derive Name Trait...)` |
| `(trait Name (method (self (p Type)...) Ret) ...)` | signatures type every call; `Self` is the implementing type; bare `(m self)` is `E_MALFORMED_FORM`; no bounds, defaults, assoc types |
| `(impl Trait Type (defn m (self ...) ...) ...)` | call `(Trait.m recv ...)`, `(recv.m ...)` or `((expr).f.m ...)`; resolved statically (a trait-generic function is specialized per type); no impl for a known type is `E_TRAIT_NOT_FOUND`, an unresolved receiver `E_CANNOT_INFER` |
| `(derive Type Trait...)` / `(derive Type [Trait...])` | generates an impl per trait: `Show` (`Name(a, b)` / `Name { f: v }`; `Secret` fields → `<secret>`, `impl-not Show` fields → `<hidden>`), `Debug` (same, strings quoted), `Eq` (`Eq.eq`, structural), `Ord` (`Ord.compare` → -1/0/1: variant declaration order, then fields lexicographically), `Hash` (`Hash.hash`, deterministic Int), `Clone` (identity). Every field type must implement the trait, no `Secret` field under Eq/Ord/Hash, only these six names — else located `E_TRAIT_NOT_DERIVABLE` |
| prelude traits (`core/show`) | `Show`, `Debug`, `Eq`, `Ord`, `Hash`, `Clone`: impls for `Int Float Bool String` and `List`/`Option`/`Result`; `Vec`/`Map` only `Show`; `StrView` all five of Show/Debug/Eq/Ord/Hash |
| `(impl-not Trait Target)` | top-level; `Target` a type or a trait (all implementors); any impl/derive of the pair, or an impl of `Trait` whose result derives from a protected value, is `E_IMPL_FORBIDDEN`; for `Show` the target gets a compiler-made `<hidden>` Show |
| prelude `(trait Secret (wipe (self) Int))` | `(impl Secret T ...)` makes `T` key material: constructors yield secrets, params/fields of `T` are secret, prints `<secret>`, `(k.wipe)` erases; prelude `(impl-not Show Secret)` and `(impl-not Debug/Eq/Ord/Hash Secret)` |
| `(extern "sym" (T...) R)` | C signature for a foreign symbol; required before `ffi-call` to it (`E_CANNOT_INFER` otherwise); concrete word-sized types only (`Int Bool String Ptr`, handles, `(Fn (A...) R)` callbacks, `Unit` result); `Float` or a type variable is `E_TYPE_MISMATCH`; retyping a `zyl_*` entry is `E_FFI_RESTRICTED` ([ffi-extern-required](../rules/ffi-extern-required.md)) |
| `(alias Name Type)` | no-op: `Name` in an annotation is then an unknown type name, i.e. a fresh type parameter, not `Type` |
| `(defmacro name (params [&rest r]) template)`, `macro` | top level only; exactly one template form (`E_MALFORMED_FORM`); `&rest r` takes the remaining arguments, spliced with `,@r` where any number of expressions may appear (call arguments, `begin`, `print`; not `(list ,@r)` or `[,@r]`, which are constructor fields: write `` `(,@r) ``) ([macro-quasiquote-and-rest](../rules/macro-quasiquote-and-rest.md)) |
| `(use path ...)`, `(pub <def>)`, `(feature-gate f <def>)` (top level only, else `E_PKG_FEATURE_NESTED`), `(module n)` (ignored), `(export n)` (dropped) | |

## Bindings and control

| Form | Notes |
|---|---|
| `(let name v body...)` / `(let (name v) body...)` | one binding; several body forms are a `begin`; `(let ((x 1) (y 2)) ...)` and a missing body are `E_MALFORMED_FORM`; local `let` is monomorphic |
| `(let-mut name v body...)`, `(set! name v)` | `set!` only on `let-mut`/`for` vars; `Unit` |
| `(fn (p...) body...)`, `(lambda ...)` | capture by value; no self-reference (a `let`-bound lambda cannot call itself) |
| `(if c t e)` | `c` is `Bool`; without `e` the form is `Unit` and `t` must be `Unit` |
| `(cond (t b...) ... (else b...))` | tested in order; `else`/`true` clause ends it; without one the `cond` is `Unit` |
| `(while c body...)` | `Unit` |
| `(for ((i 0) (j 1)) c body)`, `(for (i 0) c body)`, `(for () c body)` | no step: body must `set!`; `Unit` |
| `(begin e...)` | last value; empty → `unit` |
| `(match e arms...)` | constructor or literal arms, never mixed (not diagnosed); arms of one type |
| `(try e (catch v h...))` | catches runtime panics (`error`, `E_INDEX_OUT_OF_BOUNDS`, `E_FFI_TIMEOUT`, contract failures...), not `Err` values and not SIGFPE; handler has the body's type |
| `(with-resource (n init) body)` | binds; **no release** |
| `and`, `or`, `not` | `Bool` only; short-circuit; `(or 5 6)` and `(not 0)` are `E_TYPE_MISMATCH` |
| `(error "msg")` | lib (`allocator/allocator`), `String -> a`; panics / unwinds to `try` |
| `when`, `unless` | lib (`core/core`); **eager** functions; the body must be `Unit` and runs even when the condition says not to |
| `assert`, `unwrap` | lowered; `(assert c "msg")` panics with the literal message, else `assert failed`; `unwrap` takes an `Option` (a `Result` is `E_TYPE_MISMATCH`), `None` panics `unwrap on None` ([err-no-assert-unwrap](../rules/err-no-assert-unwrap.md)) |

## Arithmetic, comparison, bits

| Op | Notes |
|---|---|
| `+ - *` | n-ary fold left; all operands one type, `Int` or `Float`; `(- x)` negates, `(+ x)` and `(* x)` are `x` |
| `/` `%` | two operands (one is `E_ARITY_MISMATCH`); trunc toward zero; `%` sign of dividend; integer /0 → SIGFPE; Secret operand rejected; by a constant the native backend multiplies by a magic number |
| `= == != < > <= >=` | `=`≡`==`; return `Bool`; both operands one type. Strings by content everywhere (types are always known); structs/ADTs: `==`/`!=` structural (generated `T.==`); `<` etc. only on `Int`, `Float`, `String` — on a struct/ADT `E_TYPE_MISMATCH` (derive `Ord`, call `Ord.compare`) |
| `bit-and bit-or bit-xor` | n-ary, at least two operands |
| `bit-not` | exactly 1 arg |
| `shl shr ashr` | `shr` logical, `ashr` arithmetic; counts ≥ 64 defined (0 / sign fill) |

No overflow checks; bitwise ops not constant-folded; no Int/Float conversion form (`(ffi-call "zyl_f_of_int" n 1000)` converts).

## Strings

| Name | Notes |
|---|---|
| `(str-concat a b)` | inlined, fresh string |
| `(str-length s)` | bytes; inlined |
| `(str-substring s start len)` | byte-indexed fresh copy; inlined |
| `(str-equal a b)` | content, `Bool`; inlined |
| `str-eq` (`Bool`), `str-len`, `str-intern arena s`, `buf-append dst src` | lib (`allocator/allocator`) |

Zero-copy substrings: `text/view` (`StrView`, `Cursor`); see [stdlib.md](stdlib.md).

## Data

`(struct-get v "field")`, `v.field`, `v.a.b`, `(expr).field` (any expression, chained: `(seg).b.y`), `(make-Name ...)`, constructor application. `make-struct`, `make-variant`: `E_CANNOT_INFER` (write the constructor). `tuple`: undefined.

List literals build a `Cons` chain, elements evaluated left to right, all of one type (`E_TYPE_MISMATCH` otherwise; see [syn-list-literals-and-quote](../rules/syn-list-literals-and-quote.md)):

| Form | Notes |
|---|---|
| `(list a b c)`, `[a b c]` | `(list)` / `[]` is `Nil`; `[...]` is a list except as a derive's trait list |
| `(quote d)`, `'d` | constant data: numbers, strings, booleans, nested lists (`'((1 2) (3))` is `(List (List Int))`); a name inside is `E_MALFORMED_FORM` (there is no symbol type) |
| `(quasiquote d)`, `` `d `` | quote with holes: `,e` the value of `e`, `,@e` the elements of list `e` (`` `(1 ,x ,@ys) `` is `(Cons 1 (Cons x (zyl-qq-append ys Nil)))`); a name outside an unquote, a nested quasiquote, `,@` outside a list, or `,`/`,@` outside a quasiquote or macro template is `E_MALFORMED_FORM` |

## I/O

| Form | Notes |
|---|---|
| `(print e...)` | each arg on its own line; `Unit` (prints as `0` if printed); format by type: Int, Float (`%f`, six decimals), String, Bool as `1`/`0`, a type with a `Show` impl through `Show.show`; a struct/ADT **without** `Show` prints its address |
| `print-int`, `print-float`, `print-string`, `print-bool` | lib (`core/core`), typed wrappers over `print` |
| `(file-open path mode)` | fd (`Int`); -1 on failure; `mode` a literal `"r"`, `"w"`, `"a"`, optionally `+`/`b` (a variable is `E_TYPE_MISMATCH`) |
| `(file-read fd n)` → `String`, `(file-write fd text)` → `Int` (`text` a `String`), `(file-close fd)` → `Int` | |
| `read-line`, `exit`, `close` | **NL**: `read-line` is a null `String` (prints `(null)`), `exit` does not end the process, `close` is 0 |

## Actors and FFI

| Form | Notes |
|---|---|
| `(spawn entry)` | zero-parameter `fn` or named fn (parameters are `E_TYPE_MISMATCH`); immutable captures OK; returns an `Actor` |
| `(send a msg)` | `a` an `Actor`; async, FIFO per sender; one word; no-op if `a` not live; dropped if `a` never receives; `Unit` |
| `(receive)` | next data message of the running actor (or `main`); blocks; queued closure messages ahead run first; **untyped** (its result takes whatever type the use needs: the one soundness hole) |
| `(actor-self)` | running actor's `Actor`; on `main`, opens a mailbox so actors can reply |
| `(ffi-call "sym" args... timeout)` | symbol a string literal; last arg a positive integer literal timeout (ms), else compile error; foreign symbols need an `extern`, run on a worker thread and raise `E_FFI_TIMEOUT` on overrun; `zyl_*` symbols are typed by `compiler/ffi_sigs.zyl` and called directly (one with no signature is `E_CANNOT_INFER`; raw entries are `E_FFI_RESTRICTED` outside the stdlib); ≤ 16 args |
| `(ffi-pin v)` / `(ffi-unpin p)` | `a -> (Pin a)` (C gets the slot's address) / `(Pin a) -> a`; a function is `E_FFI_TYPE_NOT_PINNABLE` |
| runtime via `ffi-call` | `zyl_actor_wait_all` (`-> Unit`), `zyl_cstr_from_int arena n`, `zyl_int_text n`, `zyl_now_ms`, `zyl_argc`, `zyl_arg_str i`, `zyl_region_live_bytes`, `zyl_f_of_int`; `zyl_actor_send_closure` has no signature, so a program cannot send closure messages |

## Bytes and atomics

`(byte n)`, `(bytebuf Region cap)`, `bytebuf-cap/len/ptr`, `(byteslice buf off len)`, `(byteslice-sub s off len)`, `(bytebuf-append dst slice)`, `(load-u8 :le buf off)`, `load-i8`, `(store-u8 :le buf off v)`, `store-i8`, `load-{u,i}{16,32,64}`, `store-{u,i}{16,32,64}` (same shape), `bytebuf-atomic-{load,store,add,sub,fetch-add,max,min,cas}`, `(align-check ptr n)` (`Bool`). Handles are typed `ByteBuf` / `ByteSlice`; every offset, length and stored value is an `Int`. `byteslice`, `bytebuf-append`, `bytebuf-len/-cap/-ptr` and the atomics take a `ByteBuf`; `byteslice-sub` and `bytebuf-append`'s second operand a `ByteSlice`; a load or store takes either, and one whose handle type nothing decides is `E_CANNOT_INFER` (annotate `((b ByteBuf))`). Out-of-range accesses fail closed (0).

## Regions

`(with-region (arena :block B :align A :limit L) body)` / `(with-region (fixed :size S :align A) body)`: allocations in `body` go to a region released when `body` ends. `B` a multiple of 4096, ≤ 64 MiB; `L` byte limit (0 none); `A` a power of two 8..4096, default 8. Bad spec `E_REGION_SPEC`; out of space `E_REGION_EXHAUSTED` (catchable, compiled code only); a region value outliving `body` `E_REGION_ESCAPE`. `(ffi-call "zyl_region_live_bytes" 1000)` = bytes held by live regions. See [own-with-region](../rules/own-with-region.md).

## Contracts

`(requires C)`, `(invariant C)` (checked where written), `(ensures C)` (leading `defn` body form; checked after the body, value bound to `result`) — `C` is `Bool`; failure panics `E_CONTRACT_VIOLATION: precondition of f failed: C` (`postcondition`, `invariant`), catchable by `try`. `(recover BODY ((E_CODE) fb) ((String) fb) (_ fb))`: if `BODY` raises, arms are tried in order — an `E_` code matches by message prefix, a type-named or `_` arm matches anything, no match re-raises. `(checkpoint E)`: if `E` raises, the outer `let-mut` variables it `set!`s are restored, then the error is re-raised (byte-buffer writes are not undone). Profiles `strict`/`debug` (panic, default strict), `warn` (stderr `warning: E_CONTRACT_VIOLATION: ...`, continue), `off`/`production` (clauses compiled out): `--contracts=P` for the build, `(contracts P FORM)` for one form (lexically), bare top-level `(contracts P)` for the next form. See [contract-checks-and-profiles](../rules/contract-checks-and-profiles.md).

## Testing

`(test "name" body)` (exactly one body form; `begin` for several), `(run-tests)`, `assert-equal` (both sides one type; structural on ADTs; Floats within 1e-5), `assert-true`, `assert-false` work. `assert-fail` evaluates its operand and always passes; `test-suite` drops its tests; `setup`, `teardown`, `test-property`, `test-compile`: not run. The binary exits 0 even when a test fails.

## Types, regions, capabilities (annotation/argument names)

Types `Int Float Bool String Unit` (value `unit`), handles `Actor ByteBuf ByteSlice Arena Ptr`, constructed `List Option Result Vec Map Slice StrView (Pin a) (Array a)`, `(Fn (A...) R)` in `extern`; any other uppercase name in an annotation is a type parameter (a misspelling is not reported). Regions `Stack Heap Global Circular Pin`; capabilities `Secret` — a parameter or field annotation, or a trait (`TCap`/`TMut` are inferred). A trait name in type position is `E_MALFORMED_PARAMETER`. `declassify`, `ct-eq-bool`, `ct-eq-words-bool` drop `Secret`.
