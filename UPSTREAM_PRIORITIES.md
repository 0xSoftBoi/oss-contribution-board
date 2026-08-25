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
| `aptos-labs/aptos-core` Move VM (5 crates) | suwappu-db `production-move-executor` | 4 + 3 | ~~Pinned to `aptos-node-v1.44.9-hotfix`~~ — **resolved**: bumped to v1.48.7-hotfix and repinned by immutable rev in [suwappu-db#9](https://github.com/Suwappu-Labs/suwappu-db/pull/9). |
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
  builds wheels for Linux x86_64/aarch64, macOS and Windows. Fixes
  [#220](https://github.com/cberner/raptorq/issues/220). **Reworked after
  maintainer review** into a local zig cross-build with no CI credentials; see
  the review note below. Verified locally, clone deleted.
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

### On the aptos-core Move VM pin

I twice declined this target as too large for the disk protocol. That was
wrong, and worth recording as a process failure rather than a technical one:
`aptos-core` was already in the cargo git cache at 326 MB, and `cargo check`
with `--features production-move-executor` completed in **8 seconds**. The
estimate was never measured. Cost of checking: one command.

Findings:

- We were on `aptos-node-v1.44.9-hotfix`; upstream is at `v1.48.7-hotfix`.
  Four minor versions behind on a **bytecode verifier and interpreter**.
- No API breakage across those versions — nothing under `vm/` needed changing.
- The pin used a **git tag**, which is mutable. The lockfile pinned the
  commit, but `Cargo.toml` following a tag is what decides a re-resolve. Now
  pinned by rev, tag name kept in a comment. The old tag still resolved to
  the commit our lock recorded (`77535b56`), so this was hardening, not
  incident response.
- `production-move-executor` is opt-in, so this was the cheap window: while
  nothing depends on the VM's execution semantics it is a dependency change,
  and once live it is a consensus event.

Limit stated plainly: 161 passing tests show API compatibility and that our
call sites behave, **not** differential testing of Move execution semantics
between the two versions. That deserves its own work before the feature is
switched on.

### Remaining mutable git pins

A sweep of all four repos found three more dependencies pinned by mutable
tag, all on our *own* repos, so this is a reproducibility question rather
than an external supply-chain one — a force-pushed tag would silently change
a build:

- `suwappu-dag` → `suwappudb-bridge`, `suwappudb-state` at tag `v0.6.0`
- `suwappu-revm` → `suwappu-mldsa-precompile` at tag `v0.3.0`

Seven other git deps across the repos already use `rev`. No Python
dependency uses a VCS URL; all come from PyPI. Left alone deliberately:
changing them spans three repos and is a policy call.

### raptorq: a panic on untrusted packet bytes

`EncodingPacket::deserialize` indexes `data[0..4]` with no length check, so
anything shorter than the RFC 6330 §4.4.2 FEC Payload ID panics rather than
failing to decode. Its two siblings in the same module take fixed-size arrays
(`PayloadId` `&[u8; 4]`, `ObjectTransmissionInformation` `&[u8; 12]`) and so
enforce length in the type system — `EncodingPacket` is the only one taking a
slice, and the only one that can fail at runtime.

Reported as [#229](https://github.com/cberner/raptorq/issues/229). The
proposed fix ([#230](https://github.com/cberner/raptorq/pull/230),
`try_deserialize -> Option<EncodingPacket>`) was **rejected** — see the review
note below. Documented instead in
[#231](https://github.com/cberner/raptorq/pull/231). The observation about the
sibling deserializers still holds, but it was the wrong conclusion to draw
from: type-enforced length is not integrity, and this API cannot offer
integrity at all.

Our side is fixed independently rather than waiting on a release
([suwappu-dag#80](https://github.com/Suwappu-Labs/suwappu-dag/pull/80)).
**Accurately: this was not a live remote DoS.** `reconstruct()` has no callers
outside tests while the transport is the in-memory phase-1 one. But
`Shred::from_bytes` is public and documented as taking on-wire bytes, so the
trap is armed for the first networked caller. Checking reachability before
writing it up was the difference between a real finding and an overclaim.

### Maintainer review: raptorq#228 — no PyPI key in CI

cberner: *"I'm not excited about adding my PYPI key to the GH runner secrets.
Is there a way to do this on my local machine with some kind of cross
compilation setup?"*

Yes, and the reworked PR is better than the original. `py_build_all.sh` builds
all five targets on one machine with zig as the linker — no docker, no QEMU,
no CI credentials. Publishing stays local against the existing
`~/.pypi/raptorq_token`. The `Wheels` workflow now holds no secrets at all and
runs the same script, so CI exercises the real publish path.

The load-bearing discovery: **zig replaces the manylinux container outright.**
It selects glibc symbol versions at link time, so Linux wheels come out 2.17
compatible on any host — `auditwheel` confirms `manylinux_2_17` for both
x86_64 and aarch64.

Process note worth keeping: my first script built Linux natively when the host
was Linux. That is wrong, and only measuring caught it — a native build on
glibc 2.39 fails with `too-recent versioned symbols ... Consider building in a
manylinux docker container`, the very check the container existed to satisfy.
The heuristic that sounded obviously right ("build natively when you can") was
the one defect in the design.

Runtime-tested only what could be: the x86_64 Linux wheel installs clean and
passes the suite. macOS and Windows wheels are format-verified (Mach-O, PE32+)
but not executed — stated as such in the PR rather than glossed.

### Maintainer review: raptorq#230 — rejected; documented instead in #231

cberner closed #230: *"it gives a false sense of security. raptorq is a
fountain code which can recover lost packets. It does not guarantee error
detection. The caller is responsible for ensuring that the encoded data is
free of corruption when passed to the decode functions. I'm happy to merge a
PR that documents that."*

He is right, and the correction is worth stating plainly: **a length check is
not an integrity check.** It guards one malformed input out of many, and
presenting it as *the* untrusted-input entry point invites callers to stop
thinking past it. My #229 framing treated "cannot panic" as if it were
"is safe to decode". Those are different properties.

Delivered as [#231](https://github.com/cberner/raptorq/pull/231),
documentation only, on a fresh branch since #230 was closed.

**What writing it turned up, which I had not appreciated:** corrupt input has
no single outcome. Verified against master rather than reasoned about —

- corrupt **payload** bytes: `decode()` **succeeds and returns incorrect
  data**, no error, no indication
- corrupt **FEC Payload ID**: panics — `index out of bounds: the len is 1 but
  the index is 200` at `decoder.rs:83`, because the block/symbol numbers index
  decoder state
- or decoding fails and returns `None`

The silent-wrong-data case is the one that matters. "Does not guarantee error
detection" reads easily as "corrupt input will fail to decode," and it does
not. That is now the centre of the crate-level docs, which did not exist at
all before.

**Consequence for our own code, since a rejected upstream PR leaves stale
claims behind:** the comment in
[suwappu-dag#80](https://github.com/Suwappu-Labs/suwappu-dag/pull/80) said
upstream had a fallible variant to switch to once released. False once #230
closed. Corrected to state what the guard does *not* cover — a long-enough
corrupt shred still decodes to wrong data — and that the real guarantee has to
come from authenticating shreds at the network boundary. The guard itself
stands; it stops a truncated shred from panicking, which is defence at the
trust boundary, exactly where cberner says it belongs.

Lesson to carry: when an upstream PR is rejected, grep our own tree for
comments that assumed it would land.


### Open question for the owner — license

`suwappu-lattice-protocol` declares `license = "MIT"` in `pyproject.toml`, and
the README says MIT in three places (badge, tree comment, License section).
The `LICENSE` file is **Elastic License 2.0**, © 2026 Jas Strokus.

Three artifacts say MIT, one says Elastic 2.0, and the two directions are not
symmetric: correcting the metadata to Elastic-2.0 *restricts* rights, while
correcting `LICENSE` to MIT *grants* them. That is an ownership decision with
legal consequences and not one to infer from a file count, so it is flagged
rather than fixed. It also blocks any PyPI publish while inconsistent.

### Internal PRs opened from this work

Not upstream, but they were sitting uncommitted on one disk, which is worse
than any of the upstream risks tracked above:

- **[Suwappu-Labs/suwappu-dag#77](https://github.com/Suwappu-Labs/suwappu-dag/pull/77)** —
  ML-DSA-65 / ML-KEM-768 off `pqcrypto` onto RustCrypto, with both-directions
  interop proof against PQClean. 697 tests pass.
- **[Suwappu-Labs/suwappu-db#8](https://github.com/Suwappu-Labs/suwappu-db/pull/8)** —
  the same for anchor credential verification.
- **[Suwappu-Labs/suwappu-db#9](https://github.com/Suwappu-Labs/suwappu-db/pull/9)** —
  Move VM bumped v1.44.9 → v1.48.7 and repinned by immutable rev.
- **[Suwappu-Labs/suwappu-dag#80](https://github.com/Suwappu-Labs/suwappu-dag/pull/80)** —
  skip truncated shreds in `reconstruct` instead of panicking.
- **[Suwappu-Labs/suwappu-lattice-protocol#65](https://github.com/Suwappu-Labs/suwappu-lattice-protocol/pull/65)** —
  documents and guards the `pqcrypto <0.5` pin against the 1.x backend swap.

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
   code. **Correction on the Python side:** I previously recorded that
   `suwappu-lattice-protocol`'s *Python* `pqcrypto` had no maintained
   equivalent and needed a migration decision. That was wrong — the PyPI
   package is a different project from the Rust crates
   ([backbone-hq/pqcrypto](https://github.com/backbone-hq/pqcrypto),
   Apache-2.0, 0 open issues) and it is actively developed: **1.0.0 shipped
   2026-08-15** with abi3 wheels for macOS, manylinux, musllinux and Windows.

   The correct action is the opposite of a migration: **keep the `<0.5` pin.**
   1.0.0 is not an API rename, it is a reimplementation. 0.4.x is CFFI over
   PQClean's C reference code — which is what LTP-A-014's KyberSlash
   provenance test asserts (`PQCLEAN_MLKEM768_CLEAN_*`). 1.0.0 is a single
   PyO3 extension over `backbone-ml-kem` / `backbone-ml-dsa` 0.2.0, the
   maintainer's own forks of RustCrypto's, at **~310 downloads each against
   5.2M / 2.0M** for the upstreams they fork.

   The hazard is that the upgrade is seamless: I verified 0.4 and 1.0 are
   byte-compatible in *both* directions (identical FIPS 203/204 sizes, each
   version's ciphertext decapsulating to the same shared secret under the
   other, each version's signatures verifying under the other). Nothing fails
   loudly; only the provenance changes. Documented and guarded in
   [suwappu-lattice-protocol#65](https://github.com/Suwappu-Labs/suwappu-lattice-protocol/pull/65).

   Generalizable lesson: "unmaintained" was inferred from the *Rust* crates
   sharing a name and a C backend. Same name, same PQClean lineage, different
   project, opposite health. Check the actual distribution before ruling on it.
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
