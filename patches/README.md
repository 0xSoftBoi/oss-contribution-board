# Patches

Three ready-to-send patches for the open pull requests tracked in `index.html`.

Each was built and tested locally against the base commit named below, with GNU
coreutils 9.4 and anvil as reference implementations. Every patch is clean under
its project's own `fmt` and `clippy` settings.

Apply with `git am`.

## `alloy-4162-followup.patch`

**Apply on top of the existing PR branch, not on main.**

```sh
git fetch origin refs/pull/4162/head:pr4162 && git checkout pr4162
git am /path/to/alloy-4162-followup.patch
```

- Base: `7eb4965a` (head of [alloy#4162](https://github.com/alloy-rs/alloy/pull/4162))
- 1 file changed, +52 −25

[PR #4162](https://github.com/alloy-rs/alloy/pull/4162) is sound and its tests
pass, but it still hangs in the exact scenario
[issue #3881](https://github.com/alloy-rs/alloy/issues/3881) reports: a server
whose `eth_blockNumber` is stale and never catches up. The recheck asked that
same stale server for the chain height, so the height could never reach the
watcher's confirmation target.

A receipt in block `n` already proves the chain is at least `n` blocks long, so
the receipts supply the height themselves. The `eth_blockNumber` request is
dropped entirely — one fewer RPC call per cycle, and the block stream already
polls it.

The patch adds a regression test that mines the transaction's block while
leaving the reported block number pinned at 0. **It hangs before this change**
(`watcher hung despite the transaction being confirmed`) and passes after.

Verified: 230 `alloy-provider` lib tests pass, 0 failed. The `heart::` tests also
drop from 6.80s to 0.97s, since the recheck no longer waits on a stale response.

## `parse_datetime-j.patch`

**Branch from current main.** This replaces
[PR #286](https://github.com/uutils/parse_datetime/pull/286), which is built on a
wrong premise — see below.

```sh
git checkout -b fix/military-j main
git am /path/to/parse_datetime-j.patch
```

- Base: `46fb737` (`main`)
- 4 files changed, +177 −4

GNU accepts every letter `a`–`y` as a military time zone, and all of them are
already in the table except `j`. It is missing because it is the one letter that
does not denote a fixed offset: `j` means **local time**, so it cannot be
expressed as an `Offset` at all.

```
$ TZ=America/New_York date -d '8j'            # 08:00 -0400
$ TZ=America/New_York date -d '2026-01-01 j'  # 00:00 -0500   (DST-aware)
$ TZ=America/New_York date -d '8z'            # 04:00 -0400   (z is UTC)
```

- `j` parses as its own item, so `8j`, `8 j`, `j 8` and `8J` all work.
- Still a time zone item: `8j utc`, `8 utc j` and `8 j j` are rejected as a
  repeated zone, matching GNU.
- The whole alphabetic word is consumed before comparing, so `8jan` still parses
  as a date and `jj` is rejected rather than matching a `j` prefix.
- Unknown words stay errors — `8foo` still fails, as it does in GNU.

Verified against GNU coreutils 9.4: **all 26 letters `a`–`z` resolve to the same
instant as GNU**, in both a DST and a non-DST base zone (0 mismatches). 403 tests
pass, 0 failed.

### Why not just update #286?

`sylvestre`'s review was right. #286 assumed GNU silently discards unrecognized
trailing alphabetic tokens, but it does not — `date -d '8foo'` is an
**invalid date**. The only token GNU actually accepts there is `j`, and for a
reason #286 did not model.

#286 is also five months behind main and its diff reverts three later fixes,
including the author's own am/pm change. Close it with a note and open a fresh PR
from this patch, quoting the GNU 9.4 letter table — that shows the review was
acted on precisely.

## `coreutils-12363-rebased.patch`

**Force-push over the existing branch.**

```sh
git checkout -b fix/12312-mktemp-dotdot-prefix main
git am /path/to/coreutils-12363-rebased.patch
```

- Base: `ebb9ab1ac` (`main`)
- 2 files changed, +77

Nothing is wrong with
[PR #12363](https://github.com/uutils/coreutils/pull/12363) — it had simply
drifted 931 commits behind main. The only conflict was two test functions landing
at the same end of `tests/by-util/test_mktemp.rs`, resolved by keeping both
groups. The change itself is byte-for-byte the original commit.

Verified: all 49 mktemp tests pass, 0 failed.
