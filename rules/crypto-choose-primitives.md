# crypto-choose-primitives

> Default to ChaCha20-Poly1305 for AEAD, X25519 + HKDF for key agreement, Ed25519 for signatures, `sysrng` for key material; never use `chacharng` for keys.

## Why It Matters

The library ships safe defaults and deliberately omits dangerous options. Picking the wrong RNG or AEAD is the realistic way to misuse it.

| Need | Use | Notes |
|---|---|---|
| AEAD | `math/crypto/symmetric/chacha20poly`: `(aead-encrypt arena key nonce aad aadlen pt ptlen)` → ciphertext‖tag; `(aead-decrypt arena key nonce aad aadlen sealed ctlen)` → `(Some pt)`/`None` | no decrypt-without-verify entry point |
| AEAD (hardware) | `aesgcm`: `gcm-encrypt`/`gcm-decrypt` (+ key length arg), `gcm-seal`/`gcm-open` | requires AES-NI; `aes-available` 0 → returns `None` |
| Key agreement | `x25519-public`, `(x25519 arena priv pub)`, `x25519-checked` | |
| KDF | `(hkdf arena ikm ikmlen salt saltlen info infolen outlen)`, `hkdf-extract`/`-expand`; `pbkdf2-sha256`; `argon2id-hash` | distinct `info` labels per direction |
| Signatures | `ed25519-public-key`, `ed25519-sign`, `ed25519-verify`; ECDSA (P-256, secp256k1, P-384) with RFC 6979 nonces; RSA-PSS | |
| RSA encryption | `rsa-oaep-encrypt`/`decrypt` | keys loaded, not generated |
| Hashes | SHA-256, SHA-512, SHA3-256/512, SHAKE128/256, BLAKE2b, BLAKE3, HMAC-SHA256 | |
| Key material RNG | `math/rand/crypto`: `sysrng-fill`, `sysrng-bytes`, `sysrng-key32`, `sysrng-next-u64` (getrandom, fork-safe) | |
| Reproducible tests | `math/rand/deterministic`: `chacharng-new`, `chacharng-from-int`, ... | **never for keys** |

## Deliberately missing

PKCS#1 v1.5 (padding oracles), software AES (cache-timing), RSA key generation (too slow), randomized ECDSA nonces (nonce reuse), DER signature encoding (fixed-width r‖s only).

## Notes

- `tests/integration/math-protocol.zyl` is the reference protocol: two X25519 key pairs → HKDF into two directional keys → ChaCha20-Poly1305 message → Ed25519-signed transcript.
- Verification layers: published vectors (`--filter math`), `verify/sha2.py`, `verify/crypto.py` (vs hashlib + pyca), `verify/timing.py`.
- BLAKE3 has no SIMD backend.

## See Also

- [crypto-representations](crypto-representations.md)
- [det-nondeterminism-sources](det-nondeterminism-sources.md) - sysrng is not reproducible
