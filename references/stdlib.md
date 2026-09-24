# Standard Library Map

The stdlib is the implicit package `zyl/std`: no manifest entry, every definition visible, never capability-checked against itself. `use` takes the full path (`(use core/core)`, not `(use core)`). `core/core` is injected unless the program loads `core/core`, `core/option` or `core/result`. Loading any stdlib module exposes every loaded stdlib definition (one flat surface), so names below are taken once their module is loaded.

## core (implicit)

| Module | Definitions |
|---|---|
| `core/core` | `identity x`, `const a _b`, `flip f a b`, `compose f g`, `apply f x`, `abs`, `max`, `min`, `clamp x lo hi`, `signum`, `square`, `cube`, `xor`, `nand`, `nor`, `implies`, `when pred body`, `unless pred body` (eager!), `is-bool`, `is-zero`, `is-even`, `is-odd`, `print-int (n Int)`, `print-float (f Float)`, `print-string (s String)`, `print-bool (b Bool)`, `option-to-result opt err`, `option-from-result res default`, `result-to-option res`, `result-from-option opt err` |
| `core/option` | `(deftype Option (Some T) None)`; `option-some`, `option-none`, `option-is-some`, `option-is-none`, `option-unwrap opt default`, `option-unwrap-or`, `option-expect opt msg`, `option-map opt f`, `option-flatmap opt f`, `option-and`, `option-or`, `option-inspect` |
| `core/result` | `(deftype Result (Ok T) (Err E))`; `result-ok`, `result-err`, `result-is-ok`, `result-is-err`, `result-unwrap res default`, `result-unwrap-or`, `result-expect res msg`, `result-map res f`, `result-flatmap`, `result-and-then res f`, `result-or-else`, `result-and`, `result-or`, `result-inspect` |
| `core/list` | `(deftype List (Cons T (List T)) Nil)`; `is-nil`, `list-car`, `list-cdr`, `list-rest`, `car`, `cdr`, `cadr`, `caddr`, `cddr` (functions; `car`/`cdr` return `Option`), `list-length`, `list-append`, `list-reverse`, `list-sum` |
| `core/show` | prelude: `(trait Show (show (self) String))`, impls for Int, Float, Bool, String; List/Option/Result impls in their modules, Vec in `collections/vec`, Map in `core/map`; `print` of a Show type prints its text |
| `core/map` | `(Map String V)`, string-keyed persistent assoc map: `(deftype MapEntry (ME K V))`, `map-new`, `map-insert m k v`, `map-remove`, `map-get m k` → Option, `map-get-or m k d`, `map-has`, `map-entries`, `map-size`, `map-is-empty`, `map-map-values f m`, `map-from-list` |

## collections (arena-backed, rebind results; Vec generic, map/set Int-only)

| Module | Definitions |
|---|---|
| `collections/vec` | `(deftype Vec (VecC Int Int Int Int T))` (phantom `T`); `vec-create arena cap` (arena = `arena-create` handle, or <= 0 = new private arena, never freed; other positive Ints crash; `cap` < 0 means 0; growth doubles in the same arena), `vec-create-default cap`, `vec-len`, `vec-cap`, `vec-get v i` → T (word -1 OOB), `vec-set v i x`, `vec-push v x`, `vec-pop`, `vec-last` (word -1 empty), `vec-free`; prints `[a, b]` |
| `collections/map` | `map-create arena cap` (arena/cap as for `vec-create`), `map-create-default`, `map-len`, `map-cap`, `map-put m k v`, `map-get m k default`, `map-has`, `map-remove`, `map-find m k i len`, `map-free` |
| `collections/set` | `set-create arena cap` (as for `vec-create`), `set-len`, `set-cap`, `set-contains`, `set-add`, `set-remove`, `set-find s k i` |
| `collections/collections` | `(deftype Assoc (Empty) (AssocNode K V (Assoc K V)))`; `assoc-empty`, `assoc-put k v m`, `assoc-get k default m`, `assoc-has k m`, `assoc-remove`, `assoc-size`, `assoc-keys`, `assoc-values`, `assoc-map f m`, `assoc-fold f acc m`; `list-map f xs`, `list-filter pred xs`, `list-fold f acc xs`, `list-take n xs`, `list-drop n xs`, `list-nth n xs`, `list-set idx val xs`, `list-contains x xs`, `list-range lo hi`, `list-count pred xs` — **collection last** |

## Systems

| Module | Definitions |
|---|---|
| `allocator/allocator` | `alloc-malloc n`, `alloc-free p`, `alloc-read-int addr`, `alloc-write-int addr v`, `alloc-int v`, `alloc-incr`, `alloc-decr`, `alloc-strlen`, `arena-create block-size` (< 16 = 64 KiB default; 0 on OOM), `arena-alloc a n`, `arena-alloc-zeroed a n`, `arena-reset`, `arena-destroy`, `arena-used`, `arena-capacity`, `str-len`, `str-length`, `str-concat`, `str-substring`, `str-eq` (1/0), `str-intern arena s`, `buf-append dst src` (appends at strlen), `error msg` |
| `atomic/atomic` | `atomic-load addr`, `atomic-store`, `atomic-add`, `atomic-sub`, `atomic-max`, `atomic-min`, `atomic-cas addr expected new`, `atomic-fetch-add`, `atomic-incr`, `atomic-decr` |
| `actor/actor` | `actor-spawn`, `actor-send`, `actor-send-with-timeout` (timeout ignored; Result), `actor-is-alive`, `actor-wait`, `actor-terminate` — needs `actor` capability |
| `ffi/ffi` | `ffi-pin-value`, `ffi-unpin-value`, `ffi-safe-call`, `ffi-pin-call-unpin` (no checking) — needs `ffi` |
| `io/io` | `io-file-open-read/-write/-append`, `io-file-read h n`, `io-file-write h d`, `io-file-close`, `io-read-line`, `io-newline`, `io-print`, `io-print-int/-string/-float`, `io-safe-read/-write/-close`; `Stdout` (`make-stdout`), `StringBuffer` (`make-string-buffer`, `string-buffer-str/-len/-destroy`); trait `OutputStream` (`write`, `flush`) — needs `io` |
| `testing/testing` | `test-run`, `test-suite-run`, `assert-equal-values`, `assert-true-value`, `assert-false-value`, `assert-fail-expr`, `property-int/-bool/-string/-float` (stubs); `test-count`, `run-tests-filtered/-parallel/-with-timeout` are placeholders that `error` |

## math (≈ 7,600 lines; `(use math/math)` loads all; arena first; bytes one per word)

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

`stdlib/repl/` (`repl`, `eval`, `interp`, `reader`, `line_editor`, `highlight`, `history`, `terminal`), `stdlib/lsp/` (`lsp_server`, `json_rpc`, `vfs`, `document_manager`, `compiler_bridge`, `source_index`, `builtins`, `capability_registry`, `workspace`, `repl_integration`, `services/*`), `stdlib/compiler/` (37 modules; see [pipeline.md](pipeline.md)).
