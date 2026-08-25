# Upstream contribution priorities

Derived from what our four most-developed systems actually import, not from
what looks interesting. Reliance numbers are measured, not estimated:

- Rust: occurrences of the crate identifier across `crates/**/*.rs`
- Python: number of files containing an import of the package

Systems surveyed: `suwappubot`, `suwappu-lattice-protocol`, `suwappu-dag`,
`suwappu-db`.

## Tier 1 — load-bearing *and* structurally fragile

These are on a critical path and pinned to something that isn't a release.
An upstream problem here is our problem, and we have no fallback.

| Upstream | Where | Reliance | Why it's fragile |
|---|---|---|---|
| `crate-crypto/rust-verkle` (`banderwagon`, `ipa-multipoint`) | suwappu-db state | 24 + 2 | Pinned to a raw git rev `e27b8b4`, never released to crates.io. Powers `production-verkle`. |
| `cberner/redb` | suwappu-db state + bridge | 72 | Our entire storage substrate. Effectively single-maintainer. |
| `rustpq/pqcrypto` (`-mldsa`, `-mlkem`, `-traits`) | suwappu-dag consensus signing, lattice `crypto` extra | 6 + 1 | Pre-1.0 (0.1.x) PQ crypto on a consensus path. Lattice pins `<0.5` as a post-KyberSlash baseline. |
| `aptos-labs/aptos-core` Move VM (5 crates) | suwappu-db `production-move-executor` | 4 + 3 | Pinned to `aptos-node-v1.44.9-hotfix` — a hotfix tag, not a release. |
| `cberner/raptorq` | suwappu-dag transport (RFC 6330 shred/reconstruct) | 8 | Sole Rust implementation of RFC 6330. No alternative exists. |

## Tier 2 — largest daily surface, healthier upstreams

Contribute opportunistically; a bug here is loud but survivable.

| Upstream | Where | Reliance |
|---|---|---|
| `python-telegram-bot` | suwappubot | 75 files — the entire bot UX |
| `SQLAlchemy` | suwappubot | 68 files |
| `web3.py` | suwappubot + lattice | 38 + 5 files |
| `blake3` | dag, db, lattice | 89 + 3 |
| `tokio` / `axum` | dag, db | 221 / 53 |
| `k256` | suwappu-db bridge ECDSA | 20 |

## Tier 3 — thin-maintained edges

Small upstreams where a single unmaintained release blocks us.

- `gaiarobotics/aegis` (`aegis-shield`) — pinned to an immutable commit and
  sitting in the request path; the code comment says to repoint to a fork
  "once created". Currently an unowned dependency.
- Chain SDKs: `solders`/`solana-py` (10 + 6 files), `hyperliquid-python-sdk`
  (8), `starknet-py` (6), `tronpy` (3), `py-clob-client` (1), `pytempo`.
- `openssl-sys` — suwappu-dag has to force the `vendored` feature workspace-wide
  because `reqwest` (via `suwappudb-bridge`) breaks musl cross-compiles
  otherwise. Real friction, and upstreamable as at minimum a documentation fix.

## In flight

- **[cberner/raptorq#228](https://github.com/cberner/raptorq/pull/228)** —
  builds wheels for Linux aarch64, macOS and Windows in CI. Fixes
  [#220](https://github.com/cberner/raptorq/issues/220), open since May with no
  PR. Tier 1 upstream, self-contained change, verified locally.

## Note unrelated to upstream

`suwappu-lattice-protocol/pyproject.toml` declares `license = "MIT"` while the
repository `LICENSE` is Elastic License 2.0. The file already flags this. It
blocks any PyPI publish and is ours to fix, not an upstream's.
