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

## Working protocol

These are large Rust projects; a single `target/` runs 800MB-1GB and building
several at once has already filled the disk and taken the machine down. So:

1. Clone exactly one upstream at a time.
2. Build, verify, commit, push to the fork, open the PR.
3. Confirm the branch exists on the fork (`gh api repos/0xSoftBoi/<repo>/branches/<branch>`)
   before deleting anything.
4. `rm -rf` the clone. Then move to the next project.

Never hold two upstream working copies at once.

## In flight

- **[cberner/raptorq#228](https://github.com/cberner/raptorq/pull/228)** —
  builds wheels for Linux aarch64, macOS and Windows in CI. Fixes
  [#220](https://github.com/cberner/raptorq/issues/220). Verified locally, clone
  deleted. Awaiting maintainer "Approve and run" for first-time-contributor CI.
- **[cberner/redb#1408](https://github.com/cberner/redb/pull/1408)** —
  implements `Value`/`Key` for the ten `NonZero` integer types. Fixes
  [#873](https://github.com/cberner/redb/issues/873) for the half not blocked on
  Rust specialization. fmt + clippy clean, no-std builds, 125/125 tests pass.
  Clone deleted.
- **[cberner/redb#1409](https://github.com/cberner/redb/pull/1409)** — stacked on
  #1408; niche-encodes `Option<NonZero*>` at 4 bytes instead of 5. Closes the
  rest of #873. Clone deleted.
- **[gaiarobotics/aegis#22](https://github.com/gaiarobotics/aegis/pull/22)** —
  fixes quadratic regex backtracking in signature EV-001: a 160 KB single-line
  input cost 19.6s of CPU in one regex, with no timeout, on the scanner step
  that is the only part of `Shield.scan_input` not wrapped in `try/except`.
  Bounding the gap makes it linear (0.043s, 450x). Found by measuring, not
  reading. Clone deleted.

- **[python-telegram-bot#5339](https://github.com/python-telegram-bot/python-telegram-bot/pull/5339)** —
  `AIORateLimiter` promises a `RetryAfter` halts *all* requests, but the shared
  `_retry_after_event` was `set()` in a `finally` that runs for every request,
  so any request already in flight when the halt began released it on
  completion (0.30s into a 2s halt). A shorter backoff also released a longer
  one. Fixed by counting active backoffs. Closes
  [#5338](https://github.com/python-telegram-bot/python-telegram-bot/issues/5338).
  Two regression tests, both failing without the fix. Clone deleted.

### On rust-lang/rust#31844

The maintainer cited [specialization](https://github.com/rust-lang/rust/issues/31844)
as the blocker for the `Option<NonZero*>` niche. Contributing at that level is not
a viable path: the tracking issue has been open since February 2016 and still
carries `I-unsound`, `S-tracking-design-concerns` and `S-tracking-needs-deep-research`,
and `min_specialization` is perma-unstable and rustc-internal, so a stable-Rust
library can never use either.

The blocker dissolves instead by moving the varying behaviour onto redb's own
`Value` trait — a defaulted `niche()` method — so there is only ever one
`Option<T>` impl and nothing to specialize. That is what #1409 does. Generalizable
lesson: when an upstream cites a language feature as a blocker, check whether the
overlap can be pushed into a trait they already control.

## Queue

Next targets, hardest reliance first. One at a time, per the protocol above.

1. ~~`rustpq/pqcrypto`~~ — **resolved by migrating away, not by patching.**
   Upstream [#97](https://github.com/rustpq/pqcrypto/issues/97) announced that
   PQClean is being archived and the crates may be retired; RUSTSEC-2026-0161/
   0162/0163 already flag them unmaintained. Rather than contribute to a dying
   upstream, suwappu-dag and suwappu-db now use the pure-Rust RustCrypto
   `ml-dsa` 0.1 / `ml-kem` 0.3 crates. Wire encodings are unchanged and the
   equivalence is proved, not assumed, by `tests/pqclean_interop.rs`. pqcrypto
   remains a dev-dependency only, so the advisories no longer touch shipped
   code. **Still open:** `suwappu-lattice-protocol` uses the *Python* `pqcrypto`
   package, which has no RustCrypto equivalent — that one needs a separate
   decision (liboqs-python, or PyO3 bindings over the Rust crates).
2. `crate-crypto/rust-verkle` — **decision, not a patch.** `banderwagon`,
   `ipa-multipoint` and `verkle-trie` are *not published on crates.io at all*,
   so the raw-rev pin can never be resolved by a version bump. Upstream's last
   push was 2024-10-25 (~2 years stale, 36 open issues, not archived), which
   tracks Ethereum moving off Verkle tries. Mitigating factors: the pinned rev
   still builds clean on Rust 1.93, and `production-verkle` is default-off. So
   this is frozen-but-working. Real options are vendoring the ~26 call sites,
   or dropping the feature — not an upstream contribution.
3. ~~`gaiarobotics/aegis`~~ — pin is identical to upstream HEAD, so not
   stale; the risk is bus-factor (4 stars, third-party, in the request path).
   First contribution landed as #22 above. The fork named in suwappubot's
   `requirements.in` comment still does not exist.
4. ~~`python-telegram-bot`~~ — first contribution landed as #5339 above. Our pin
   is 22.8 with the `[rate-limiter]` extra, so the flood-halt bug was on our own
   send path. Note the existing `test_delay_all_pending_on_retry` looked like it
   covered this: it starts its second request *after* the halt, so that request
   blocks harmlessly at the internal `wait()`. The bug only appears for a request
   already past that wait, which needs the 429 to arrive after some latency —
   i.e. the test was shaped so the defect could not show up. Worth remembering:
   an existing test asserting the guarantee is not evidence the guarantee holds.
5. `SQLAlchemy` — 68 files in suwappubot, not yet surveyed.

## Note unrelated to upstream

`suwappu-lattice-protocol/pyproject.toml` declares `license = "MIT"` while the
repository `LICENSE` is Elastic License 2.0. The file already flags this. It
blocks any PyPI publish and is ours to fix, not an upstream's.
