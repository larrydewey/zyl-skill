# pkg-mvs-lock-store

> Upgrade by editing a requirement; commit `zyl.lock`; build CI with `zyl build --locked`; run `zyl fetch` (the only networked command) before building registry deps.

## Why It Matters

**MVS**: each package's selected version is the greatest minimum any manifest requires, within one compatibility unit (a major; each 0.x minor is its own unit). No search, no backtracking, no index consultation. Adding a dependency never silently upgrades another. Unsatisfiable within a major: `E_PKG_VERSION_CONFLICT`. Pre-releases are selected only when named exactly.

**Lock** (`zyl.lock`): an integrity/provenance record, **not** a resolution input (deleting it changes nothing about selection). Written by `zyl fetch`/`zyl update`; `zyl build` only reads it. Records version, source, BLAKE3 archive hash, pinned Ed25519 key, signature, features, capabilities, capability closure, graph hash.

**Store**: `~/.zyl/store/blake3/<hash>/` (or `$ZYL_HOME`). `zyl build` is offline; a missing package is `E_PKG_NOT_IN_STORE`.

## `--locked` failures

| Condition | Error |
|---|---|
| no lock / graph differs | `E_PKG_LOCK_STALE` |
| capability closure grew | `E_PKG_CAPABILITY_GROWTH` |
| malformed / unknown lock version | `E_PKG_LOCK_INVALID` |

## Trust

Publisher signs the archive's BLAKE3 hash with Ed25519; verification is mandatory. First resolution pins the key (lock + `~/.zyl/keys/`). Key change `E_PKG_KEY_CHANGED`, hash change `E_PKG_HASH_MISMATCH`, unsigned `E_PKG_UNSIGNED`, bad sig `E_PKG_SIGNATURE_INVALID`, yanked (new resolutions only) `E_PKG_YANKED`.

## Current status

- The default index URL is a placeholder; `zyl fetch`/`update` need a local git clone at `~/.zyl/index` even for path-only graphs, else `E_PKG_FETCH_FAILED`.
- `git` deps are recognized but not cloned by `zyl fetch`.
- MVS does not check a path dep's requirement against its manifest version outside a workspace.
- Path deps record no hash/key/signature.
- `zyl vendor` copies the graph but nothing reads `./vendor`; no build cache.
- `zyl publish` leaves the archive in `~/.zyl/tmp/` with `(url "https://REPLACE-ME")`.

## See Also

- [pkg-manifest](pkg-manifest.md)
- [det-nondeterminism-sources](det-nondeterminism-sources.md)
