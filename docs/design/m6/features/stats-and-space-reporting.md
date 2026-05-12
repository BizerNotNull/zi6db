# M6 Feature Plan: Stats And Space Reporting

This document defines the concrete M6 implementation plan for the
`stats and space reporting` feature referenced by
[docs/design/m6/m6-implementation-plan.md](/D:/代码D/zi6db/docs/design/m6/m6-implementation-plan.md).
It follows the repository rules in
[AGENTS.md](/D:/代码D/zi6db/AGENTS.md) and narrows the broad M6 milestone into
one implementable surface.

The purpose of this feature is to expose stable, snapshot-aware accounting for:

- logical database size from selected metadata
- physical file length when it differs from logical size
- reusable versus pending free space
- exact versus estimated counters
- transaction-pinned stats for readers that outlive newer commits

This feature must not introduce a second source of truth for page ownership.
All space reporting must derive from the same metadata, freelist, reader
registry, and page walkers used by M6 overflow and reuse logic.

## 1. Scope

M6 stats and space reporting is complete when:

- callers can request database-level stats from the current selected metadata
- callers can request transaction-pinned stats from a read or write transaction
- reusable and pending space are reported separately and clearly
- logical file size is distinguishable from physical file size
- every field documents whether it is exact or an estimate
- stats remain stable within a pinned read transaction
- tests prove stats behavior across commit, reopen, reader pinning, promotion,
  and reuse

M6 does not promise:

- bucket-level statistics
- background aggregation threads
- file shrinking or vacuum-style compaction
- expensive full-file verification on every stats call

## 2. Public API Shape

M6 should expose one shared stats shape for both database-level and
transaction-level callers.

Recommended Zig surface:

```zig
pub const DBStats = struct {
    page_size: u32,
    page_count: u64,
    file_size_bytes: u64,
    physical_file_size_bytes: ?u64,

    metadata_pages: u64,
    freelist_pages_exact: u64,
    branch_pages_estimate: ?u64,
    leaf_pages_estimate: ?u64,
    overflow_pages_estimate: ?u64,

    reusable_pages: u64,
    pending_pages: u64,
    free_pages_total: u64,
    used_pages_estimate: u64,

    reusable_bytes: u64,
    pending_bytes: u64,
    free_bytes_total: u64,
    used_bytes_estimate: u64,

    largest_reusable_run_pages: u64,
    active_read_transactions: u64,
    oldest_reader_txn_id: ?u64,

    snapshot_txn_id: u64,
    is_tx_pinned: bool,
};

pub fn stats(db: *DB) !DBStats;
pub fn statsTx(tx: *const Tx) !DBStats;
```

Field naming may adjust to the final Zig style used by the codebase, but the
semantics should stay fixed:

- exact fields must either always contain an exact value or be omitted
- estimated fields must either include `estimate` in the field name or use an
  optional field documented as approximate
- `stats(db)` reports the current selected DB snapshot
- `statsTx(tx)` reports the transaction's pinned snapshot and must not consult
  newer metadata after the transaction begins

## 3. Field Semantics

### 3.1 Required Exact Fields

The following fields must be exact in M6:

- `page_size`
- `page_count`
- `file_size_bytes`
- `metadata_pages`
- `freelist_pages_exact`
- `reusable_pages`
- `pending_pages`
- `free_pages_total`
- `reusable_bytes`
- `pending_bytes`
- `free_bytes_total`
- `largest_reusable_run_pages`
- `active_read_transactions`
- `oldest_reader_txn_id`
- `snapshot_txn_id`
- `is_tx_pinned`

Exact-count rules:

- `file_size_bytes = page_count * page_size`
- `free_pages_total = reusable_pages + pending_pages`
- `reusable_bytes = reusable_pages * page_size`
- `pending_bytes = pending_pages * page_size`
- `free_bytes_total = free_pages_total * page_size`

### 3.2 Estimated Fields

The following fields may be estimates in M6 unless the implementation already
has exact transactional counters or a cheap full traversal:

- `branch_pages_estimate`
- `leaf_pages_estimate`
- `overflow_pages_estimate`
- `used_pages_estimate`
- `used_bytes_estimate`

Estimate rules:

- an estimated field must never be presented as exact
- `used_pages_estimate` should default to `page_count - free_pages_total`
- `used_bytes_estimate` should default to `used_pages_estimate * page_size`
- if M6 later adds exact live-page traversal, the implementation may add
  separate exact fields without breaking the estimate semantics above

### 3.3 Metadata Pages

M6 should report `metadata_pages` as the fixed count of reserved non-reusable
pages at the front of the file:

- page `0`: header
- page `1`: metadata slot A
- page `2`: metadata slot B

The value is therefore expected to remain `3` for all normal M6 databases.

## 4. Logical Versus Physical Size

Stats must distinguish the logical database size from the underlying file
length on disk.

Use these terms consistently:

- `page_count`: exclusive upper bound of page IDs reachable from selected
  metadata
- `file_size_bytes`: logical size implied by `page_count`
- `physical_file_size_bytes`: optional OS-reported file length

M6 reporting rules:

- `file_size_bytes` is the user-facing size baseline for all normal accounting
- `physical_file_size_bytes` is diagnostic and may be `null` if the project
  chooses not to expose OS file length yet
- ignored tail pages beyond selected `page_count` must not be counted as used,
  reusable, pending, or allocated
- failed commits, abandoned tail growth, and ambiguous pre-metadata writes may
  increase `physical_file_size_bytes` without changing `file_size_bytes`

This distinction matters because M6 guarantees safe reuse inside the selected
logical file image, not physical shrinkage or automatic tail reclamation.

## 5. Data Sources And Computation Order

Stats must build from existing M6 ownership sources in this order:

1. Read the selected metadata snapshot.
2. Resolve the selected freelist page or chain from that snapshot.
3. Decode exact reusable and pending counts from the freelist image.
4. Read reader-registry facts from the M3 registry seam.
5. Optionally attach estimated live-page counters derived from maintained
   in-memory counters or a bounded traversal path.

Required boundaries:

- `stats(db)` reads the DB's current selected metadata and freelist snapshot
- `statsTx(tx)` reads `tx.snapshot` fields captured at transaction start
- read-only stats calls must not promote pending groups or mutate freelist state
- stats must not infer free space by scanning unreachable pages outside the
  selected freelist image

## 6. Tx-Pinned Stats

Transaction-pinned stats are required for M6 because readers can outlive newer
writer commits.

Implementation requirements:

- `statsTx(tx)` must use the transaction's pinned `root_page_id`,
  `freelist_page_id`, `page_count`, and `snapshot_txn_id`
- a read transaction opened before a writer commit must continue reporting the
  old `page_count`, old free-space totals, and old oldest-visible space state
- a write transaction may report its starting snapshot until commit publishes
  new metadata; M6 should avoid exposing speculative post-commit counters from
  uncommitted writer-private state through `statsTx(tx)`
- `active_read_transactions` and `oldest_reader_txn_id` should still reflect the
  current process registry, even when the page/freelist snapshot is old

This keeps the API honest: page accounting is snapshot-pinned, while reader
registry facts describe current in-process visibility constraints.

## 7. Reusable Versus Pending Space

M6 user-facing docs and comments must clearly separate these states:

- `reusable_pages`: free pages immediately allocatable by the next writer
- `pending_pages`: pages already freed logically but still blocked by older
  readers
- `free_pages_total`: `reusable_pages + pending_pages`

Documentation requirements:

- never describe `pending_pages` as available space
- explain that pending pages are reclaimable only after an older-reader cutoff
  is satisfied and that the promotion becomes durable only through a later
  committed freelist image
- keep the term `pending_free` for internal and design-level discussion so it
  matches the rest of M6

## 8. Implementation Steps

### 8.1 Define The Stats Shape

- add `DBStats` to the public API layer or the smallest stable internal module
  that already owns database-facing metadata types
- freeze field names and comments before wiring call sites
- prefer explicit `*_estimate` suffixes over prose-only warnings in comments

### 8.2 Add Snapshot-Aware Builders

- implement one internal builder that accepts a selected snapshot descriptor
- call that builder from both `stats(db)` and `statsTx(tx)`
- require the builder input to include `page_count`, `freelist_page_id`,
  `snapshot_txn_id`, and a flag indicating whether the caller is transaction
  pinned

### 8.3 Decode Exact Freelist Counts

- compute `reusable_pages` from persisted reusable ranges
- compute `pending_pages` from all persisted `pending_free` groups
- compute `largest_reusable_run_pages` from the same decoded reusable ranges
- compute `freelist_pages_exact` from the freelist page chain length in the
  selected snapshot

### 8.4 Attach Reader Registry Facts

- use the M3 reader registry seam for `active_read_transactions`
- use the same registry query that freelist promotion logic trusts for
  `oldest_reader_txn_id`
- do not add a second parallel count cache for stats only

### 8.5 Attach Optional Live-Page Estimates

- if the implementation already maintains exact counters for branch, leaf, or
  overflow pages in commit metadata, expose them through exact fields or rename
  accordingly
- otherwise provide estimated fields only and document their derivation
- do not walk every live page on every normal stats call unless the repo
  deliberately accepts that cost

### 8.6 Expose Logical And Physical Sizes

- always compute `file_size_bytes` from logical `page_count`
- optionally query the file length for `physical_file_size_bytes`
- keep physical size collection failure non-fatal if the logical snapshot is
  otherwise valid

## 9. Test Plan

### 9.1 Shape And Invariant Tests

- empty database stats report `page_size`, `page_count`, `metadata_pages`,
  `free_pages_total`, and byte totals consistently
- `file_size_bytes = page_count * page_size`
- `free_pages_total = reusable_pages + pending_pages`
- `free_bytes_total = reusable_bytes + pending_bytes`
- estimate fields are either present with estimate semantics or omitted cleanly

### 9.2 Logical Versus Physical Size Tests

- a failed or interrupted tail-growth path leaves `file_size_bytes` unchanged
  when selected metadata did not advance
- if `physical_file_size_bytes` is exposed, it may exceed `file_size_bytes`
  after ignored tail pages are present
- ignored tail pages do not increase reusable, pending, or used logical counts

### 9.3 Reader-Pinned Stats Tests

- open a read transaction, commit a writer change, then confirm `statsTx(read)`
  still reports the pre-commit snapshot
- confirm `stats(db)` after the same writer commit reports the newer snapshot
- confirm a read transaction pinned before a delete still reports old free-space
  counts while `stats(db)` can show the newer pending state

### 9.4 Pending And Reusable Transition Tests

- delete large data with an old reader active and confirm `pending_pages`
  increases while `reusable_pages` does not
- close the blocking reader, run the next writer promotion point, and confirm
  `reusable_pages` increases only after the committed freelist update
- confirm stats calls alone do not trigger pending promotion

### 9.5 Reuse Tests

- free pages, then perform a later write and confirm `reusable_pages` falls as
  pages are consumed
- confirm `page_count` stays stable when reuse is sufficient
- confirm `largest_reusable_run_pages` reflects fragmentation changes after
  freeing and reusing overflow chains

### 9.6 Reopen And Recovery Tests

- after clean reopen, stats preserve freelist-derived counts
- after recovery from a failed commit, stats follow the selected metadata slot
  rather than the physical tail
- corrupted selected freelist pages fail stats collection through the same
  corruption path used by open or transaction reads

## 10. M7 Handoff

This feature should leave M7 with stable accounting seams rather than forcing a
stats redesign.

Required handoff points:

- one internal snapshot-based stats builder that M7 verify and backup code can
  reuse for pinned accounting
- stable separation between logical `file_size_bytes` and diagnostic
  `physical_file_size_bytes`
- stable `pending_pages` versus `reusable_pages` terminology for future
  compaction reporting
- one reader-registry query path for `oldest_reader_txn_id`
- a clear boundary where exact page-class counters could later replace M6
  estimates without changing snapshot semantics

M7 may add:

- verify-time exact live-page walks
- backup accounting built on pinned read transactions
- compaction before/after reporting

M6 should not pre-commit to those deeper costs in the normal `stats()` path.

## 11. Acceptance Criteria

This feature is complete for M6 when:

- `stats(db)` and `statsTx(tx)` are both defined or their exact equivalents are
  exposed with the same semantics
- `DBStats` clearly distinguishes exact fields from estimate fields
- logical `file_size_bytes` is defined from selected metadata, not OS file
  length
- physical file length, if exposed, is documented as diagnostic only
- `reusable_pages`, `pending_pages`, and `free_pages_total` are exact and
  mutually consistent
- stats remain stable inside a pinned read transaction across later writer
  commits
- stats calls do not mutate pending promotion or freelist state
- tests cover empty state, large insert, delete with reader pinning, promotion,
  reuse, reopen, and ignored physical tail cases
- terminology matches the wider M6 plan: `pending_free`, `reusable`,
  `page_count`, `file_size_bytes`, and `free_pages_total`
