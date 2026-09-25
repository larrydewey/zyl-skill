# Standard Library Map

The stdlib is the implicit package `zyl/std`: no manifest entry, every definition visible, never capability-checked against itself. `use` takes the full path (`(use core/core)`, not `(use core)`). `core/core` is injected unless the program loads `core/core`, `core/option` or `core/result`. Loading any stdlib module exposes every loaded stdlib definition (one flat surface), so names below are taken once their module is loaded. Every definition is type-checked like user code; the raw runtime entries some of them wrap (`zyl_word_*`, `zyl_view_*`, `zyl_ptr_*`, ...) are `E_FFI_RESTRICTED` in user code, so call the library function instead.

## core (implicit)

| Module | Definitions |
|---|---|
| `core/core` | `identity x`, `const a _b`, `flip f a b`, `compose f g`, `apply f x`, `abs`, `max`, `min`, `clamp x lo hi`, `signum`, `square`, `cube`, `xor`, `nand`, `nor`, `implies`, `when pred body`, `unless pred body` (eager functions; `body` must be `Unit`), `is-bool`, `is-zero`, `is-even`, `is-odd`, `print-int (n Int)`, `print-float (f Float)`, `print-string (s String)`, `print-bool (b Bool)` (prints `1`/`0`), `option-to-result opt err`, `option-from-result res default`, `result-to-option res`, `result-from-option opt err` |
| `core/option` | `(deftype Option (Some T) None)`; `option-some`, `option-none`, `option-is-some`, `option-is-none`, `option-unwrap opt default`, `option-unwrap-or`, `option-expect opt msg`, `option-map opt f`, `option-flatmap opt f`, `option-and`, `option-or`, `option-inspect` |
| `core/result` | `(deftype Result (Ok T) (Err E))`; `result-ok`, `result-err`, `result-is-ok`, `result-is-err`, `result-unwrap res default`, `result-unwrap-or`, `result-expect res msg`, `result-map res f`, `result-flatmap`, `result-and-then res f`, `result-or-else`, `result-and`, `result-or`, `result-inspect` |
| `core/list` | `(deftype List (Cons T (List T)) Nil)`; `is-nil`, `list-car`, `list-cdr`, `list-rest`, `car`, `cdr`, `cadr`, `caddr`, `cddr` (functions; `car`/`cdr`/`list-car`/`list-cdr` return `Option`), `list-length`, `list-append`, `list-reverse`, `list-sum`, `list-compare`, `list-hash`; `zyl-qq-append` (what `,@` in a quasiquote expands to) |
| `core/show` | prelude traits `Show`, `Debug`, `Eq`, `Ord`, `Hash`, `Clone` (`(trait Show (show (self) String))`, `(trait Ord (compare (self (other Self)) Int))`, ...) and `Secret` (`wipe`), impls for Int, Float, Bool, String; List/Option/Result impls in their modules, Vec in `collections/vec`, Map in `core/map`; `print` of a Show type prints its text |
| `core/map` | `(Map String V)`, a string-keyed persistent association list (`str-eq` keys): `(deftype MapEntry (ME K V))` (`ME.key`, `ME.value`), `map-new`, `map-insert m k v`, `map-remove`, `map-get m k` → Option, `map-get-or m k d`, `map-has` (Int 1/0), `map-entries`, `map-size`, `map-is-empty`, `map-map-values f m`, `map-from-list` |

## text

| Module | Definitions |
|---|---|
| `text/view` | zero-copy substrings. `(deftype StrView (StrViewC String Int Int))` (base, offset, length; Show/Debug/Eq/Ord/Hash); bounds checked once when made: `view-of s`, `view-slice s off len`; then `view-sub v off len`, `view-drop v n`, `view-take v n` (clamped), `view-len`, `view-is-empty`, `view-byte-at v i`, `view-to-string` (the one copy), `view-compare`, `view-eq`, `view-eq-str v s`, `view-find v byte from` (-1 if absent), `view-starts-with v s`, `view-ends-with v s`, `view-split v byte` → `(List StrView)`, `view-trim`, `view-trim-start`, `view-trim-end`, `view-parse-int` → `(Option Int)`. Parsing cursor: `(deftype Cursor (CursorC StrView Int))`, `cursor-new v`, `cursor-of s`, `cursor-view`, `cursor-pos`, `cursor-at-end`, `cursor-rest`, `cursor-peek`, `cursor-advance c n`, `cursor-take-while c pred` → `Taken` (`taken-view`, `taken-rest`), `cursor-skip-space`, `cursor-expect c s` → `(Option Cursor)`. A view holds its base, so region inference keeps the base alive as long as any view ([data-views-and-slices](../rules/data-views-and-slices.md)) |

## collections (arena-backed, rebind results; Vec generic, map/set Int-only)

| Module | Definitions |
|---|---|
| `collections/vec` | `(deftype Vec (VecC (Array T) Int Arena))` (typed runtime array, length, arena); `vec-create (arena Arena) (cap Int)` (an `Arena` from `arena-create`; an Int such as 0 is `E_TYPE_MISMATCH`; `cap` < 0 means 0; growth doubles in the same arena), `vec-create-default cap` (new private arena, never freed), `vec-len`, `vec-cap`, `vec-get v i` (outside the Vec `E_INDEX_OUT_OF_BOUNDS`), `vec-get-or v i default`, `vec-set v i x` (index `len` within capacity extends; other out-of-range indices leave it unchanged), `vec-push v x`, `vec-pop` (empty stays empty), `vec-last` (empty `E_INDEX_OUT_OF_BOUNDS`), `vec-free`; `array-new`, `array-get`, `array-set`, `array-cap`; prints `[a, b]`. A unique, dead Vec is updated in place by the reuse pass |
| `collections/slice` | zero-copy windows on a Vec's storage: `(deftype Slice (SliceC (Array T) Int Int))`; `slice-of-vec v`, `slice-vec v off len`, `slice-sub s off len`, `slice-get s i` (range errors `E_INDEX_OUT_OF_BOUNDS`), `slice-get-or s i default`, `slice-drop`, `slice-take` (clamped), `slice-len`, `slice-fold s f init`, `slice-to-vec s arena`; Show. A slice shares storage: a later `vec-set` on a Vec using that storage is visible through it |
| `collections/map` | `(defstruct Map ...)`, Int keys and values: `map-create arena cap` (arena/cap as for `vec-create`), `map-create-default cap`, `map-len`, `map-cap`, `map-put m k v`, `map-get m k default`, `map-has` (Bool), `map-remove`, `map-find m k i len`, `map-free`. `core/map` defines another type named `Map` ([boot-one-deftype-per-name](../rules/boot-one-deftype-per-name.md)) |
| `collections/set` | `(defstruct Set ...)`, Int elements: `set-create arena cap`, `set-create-default cap`, `set-len`, `set-cap`, `set-contains`, `set-add`, `set-remove`, `set-find s k i` |
| `collections/collections` | `(deftype Assoc (Empty) (AssocNode K V (Assoc K V)))`; `assoc-empty`, `assoc-put k v m`, `assoc-get k default m`, `assoc-has k m`, `assoc-remove`, `assoc-size`, `assoc-keys`, `assoc-values`, `assoc-map f m`, `assoc-fold f acc m`; `list-map f xs`, `list-filter pred xs`, `list-fold f acc xs`, `list-take n xs`, `list-drop n xs`, `list-nth n xs` → Option, `list-set idx val xs`, `list-contains x xs`, `list-range lo hi`, `list-count pred xs` — **collection last** |

## Systems

| Module | Definitions |
|---|---|
| `allocator/allocator` | `alloc-malloc n`, `alloc-free p`, `alloc-read-int addr`, `alloc-write-int addr v`, `alloc-offset p n`, `alloc-cstr p`, `alloc-int v`, `alloc-incr`, `alloc-decr`, `alloc-strlen`, `arena-create block-size` → `Arena` (< 16 = 64 KiB default), `arena-alloc a n`, `arena-alloc-zeroed a n`, `arena-reset`, `arena-destroy`, `arena-used`, `arena-capacity`, `str-len`, `str-length`, `str-concat`, `str-substring`, `str-eq` (Bool), `str-intern arena s`, `buf-new arena n`, `buf-str b`, `buf-append dst src` (appends after the StrBuf's recorded length; past its capacity it panics `E_INDEX_OUT_OF_BOUNDS`; `arena-alloc` gives a `Ptr`, not a StrBuf), `error msg` (`String -> a`) |
| `atomic/atomic` | `atomic-load addr`, `atomic-store`, `atomic-add`, `atomic-sub`, `atomic-max`, `atomic-min`, `atomic-cas addr expected new`, `atomic-fetch-add`, `atomic-incr`, `atomic-decr` |
| `actor/actor` | `actor-spawn`, `actor-send`, `actor-send-with-timeout` (timeout ignored; Result), `actor-is-alive`, `actor-wait`, `actor-terminate` — needs `actor` capability |
| `ffi/ffi` | `ffi-pin-value`, `ffi-unpin-value`, `ffi-safe-call`, `ffi-pin-call-unpin` (no checking) — needs `ffi` |
| `io/io` | `io-file-open-read/-write/-append`, `io-file-read h n`, `io-file-write h d`, `io-file-close`, `io-read-line`, `io-newline`, `io-print`, `io-print-int/-string/-float`, `io-safe-read/-write/-close`; `Stdout` (`make-stdout`), `StringBuffer` (`make-string-buffer`, `string-buffer-str/-len/-destroy`); trait `OutputStream` (`write`, `flush`) — needs `io` |
| `testing/testing` | `test-run`, `test-suite-run`, `assert-equal-values`, `assert-true-value`, `assert-false-value`, `assert-fail-expr`, `property-int/-bool/-string/-float` (stubs); `test-count`, `run-tests-filtered/-parallel/-with-timeout` are placeholders that `error` |

## math (≈ 7,650 lines; `(use math/math)` loads all; arena first; bytes one per word)

| Module | Entry points |
|---|---|
| `math/bits` | `add32 sub32 mul32 shl32 shr32 rotl32 rotr32 not32 rotl64 rotr64 not64`, `u64-lt/gt/le/ge`, `byte-of`, `be-pack32`, `le-pack32`, masks |
| `math/words` | `w-alloc arena n`, `w-get`, `w-set`, `w-fill`, `w-copy`, `w-from-string`, `w-from-hex`, `w-hex-bytes`, `w-hex-words` |
| `math/secret/secret` | `ct-mask`, `ct-is-zero`, `ct-is-nonzero`, `ct-select`, `ct-eq`, `ct-ne`, `ct-eq-words`, `ct-ne-words`, `ct-eq-bool`, `ct-eq-words-bool`, `declassify`, `zeroize base n`, `zeroize-bytes` — needs `secret` capability in packages |
| `math/bignum/*` | `bignum`: `bn-alloc bn-add bn-sub bn-mul bn-sqr bn-eq bn-lt bn-from-bytes-be/le bn-to-bytes-be/le bn-bit-length`, `bn-cond-copy`; `montgomery`: `mont-mul mont-to mont-from mont-exp`; `barrett-reduce`; `modular`: `mod-add mod-sub mod-inverse-prime mr-probably-prime` (24-bit limbs, LS first) |
| `math/rand/*` | `rand`: `rand-u64-from-bytes`, `rand-below`; `crypto` (getrandom): `sysrng-fill sysrng-bytes sysrng-next-u64 sysrng-key32`; `deterministic` (ChaCha20, tests only): `chacharng-new chacharng-from-int chacharng-bytes chacharng-fill chacharng-next-u64 chacharng-below` |
| `math/hash/*` | `sha2`: `sha256-words`, `sha256-bytes arena msg len`, `sha256-hex-of-string`; `sha512`: `sha512-bytes`, `sha512-hex-of-string`; `sha3`: `sha3-256-bytes sha3-512-bytes shake128-bytes shake256-bytes`; `blake2b`: `blake2b blake2b-512 blake2b-hex-of-string`; `blake3`: `blake3-hash arena msg len outlen`, `blake3-hex-of-string`; `hmac`: `hmac-sha256 hmac-sha256-verify` |
| `math/crypto/symmetric/*` | `chacha20-block chacha20-xor`; `poly1305-mac poly1305-verify`; `chacha20poly`: `aead-encrypt aead-decrypt`; `aesgcm`: `aes-available gcm-encrypt gcm-decrypt gcm-seal gcm-open` (AES-NI only) |
| `math/crypto/asymmetric/*` | `x25519 x25519-public x25519-basepoint x25519-checked`; `ed25519-public-key ed25519-sign ed25519-verify`; `ecdsa`: `ec-p256 ec-secp256k1 ec-p384 ecdsa-public-key ecdsa-sign ecdsa-verify`; `rsa`: `rsa-public-key rsa-private-key rsa-oaep-encrypt/decrypt rsa-pss-sign/verify` |
| `math/crypto/kdf/*` | `hkdf hkdf-extract hkdf-expand`; `pbkdf2-sha256`; `argon2id-hash` |

## Tooling libraries

`stdlib/repl/` (`repl`, `eval`, `interp`, `reader`, `line_editor`, `highlight`, `history`, `terminal`), `stdlib/lsp/` (`lsp_server`, `json_rpc`, `lsp_types`, `vfs`, `document_manager`, `compiler_bridge`, `source_index`, `builtins`, `capability_registry`, `workspace`, `repl_integration`, `services/*`: completion, hover, goto_definition, document_symbols, semantic_tokens, signature_help, inlay_hints, call_hierarchy, code_action), `stdlib/compiler/` (41 modules; see [pipeline.md](pipeline.md)), `stdlib/mlib/deep.zyl` (a compiler stress fixture, not a library).
