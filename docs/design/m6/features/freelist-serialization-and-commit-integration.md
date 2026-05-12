# M6 Feature Plan: Freelist Serialization And Commit Integration

This document defines the concrete M6 implementation plan for freelist
serialization and commit integration. It refines
[`docs/design/m6/m6-implementation-plan.md`](/D:/代码D/zi6db/docs/design/m6/m6-implementation-plan.md),
inherits the crash and commit boundaries from
[`docs/design/m4/m4-implementation-plan.md`](/D:/代码D/zi6db/docs/design/m4/m4-implementation-plan.md),
and follows the repository rules in
[`AGENTS.md`](/D:/代码D/zi6db/AGENTS.md).

The goal is to turn the M0 freelist model into a precise, durable M6 feature:

- `reusable` and `pending_free` must serialize without loss
- freelist contents may span multiple pages
- freelist pages must account for themselves during copy-on-write commit
- metadata remains the only publication switch
- crash recovery must trust only the freelist reachable from selected metadata
- tests must prove reuse safety, validation, and crash behavior

## Goal

This feature is complete when:

- the engine can serialize and deserialize exact `reusable` and
  `pending_free[freeing_txn_id]` state
- a freelist image can span more than one `PAGE_TYPE_FREELIST` page
- commit can replace the freelist image without reusing the same transaction's
  freed pages
- old freelist pages are handed off into `pending_free[current_txn_id]`
- validation rejects malformed, overlapping, duplicated, or self-referential
  freelist entries
- crash recovery selects either the old freelist image or the new one through
  normal metadata selection, never a mixed state

## Scope

This feature covers:

- on-disk encoding for `reusable` ranges and `pending_free` groups
- multi-page freelist image layout and chaining
- transaction-local freelist assembly before commit
- freelist self-accounting during copy-on-write replacement
- metadata handoff from the commit path to the selected freelist root page
- validation at decode, commit-finalization, and checker boundaries
- crash, recovery, corruption, and space-reuse tests

This feature does not cover:

- overflow value encoding itself
- compaction, truncation, or vacuum
- background reclamation
- cross-process reader cutoff persistence beyond the M6 single-process model
- repair of malformed freelist pages

## Required Invariants

The implementation must keep these rules explicit:

1. The freelist reachable from selected metadata is the only authoritative free
   space state.
2. `reusable` and `pending_free` are disjoint.
3. A page ID may appear in at most one committed owner class:
   reachable tree/overflow content, `reusable`, or one pending group.
4. Pages freed by the active writer remain under
   `pending_free[current_txn_id]` and are never reused by that same commit.
5. Newly allocated freelist pages are not listed in the freelist image they
   store.
6. Old freelist pages replaced by a new image are added to
   `pending_free[current_txn_id]`.
7. Metadata publication is the only point where the new freelist root becomes
   visible.
8. Recovery must not infer free pages from file holes, ignored tail pages, or
   unreachable normal pages.

## Data Model

The persisted freelist should mirror the logical in-memory model:

- `reusable`: sorted non-overlapping page ranges immediately allocatable by a
  later writer
- `pending_free`: sorted groups keyed by `freeing_txn_id`, where each group
  stores sorted non-overlapping page ranges

Recommended canonical rules:

- represent singleton pages as one-page ranges
- keep ranges sorted by `start_page_id`
- merge adjacent or overlapping ranges within the same section before encoding
- keep pending groups sorted by increasing `freeing_txn_id`
- keep the overall image deterministic so tests and corruption checks do not
  depend on map iteration order

The implementation should expose one internal canonical image structure that is
used by allocator setup, serialization, deserialization, validation, and
checker traversal.

## On-Disk Encoding

### Section Layout

Bind the M0 freelist reservation to a sectioned body format:

1. freelist page header in the common normal-page header
2. freelist body header with:
   - `freelist_sequence_index`
   - `freelist_page_count`
   - `next_freelist_page_id`
   - `reusable_range_count_total`
   - `pending_group_count_total`
   - reserved bytes set to zero
3. page-local item area containing encoded records

Encode logical records as:

- `reusable_range`:
  - `start_page_id`
  - `page_count`
- `pending_group_header`:
  - `freeing_txn_id`
  - `range_count`
- `pending_group_range`:
  - `start_page_id`
  - `page_count`

The exact byte widths should match the M0 page-id and transaction-id widths.
The format should avoid variable interpretation rules that depend on parser
state outside the current logical record stream.

### Reusable And Pending Encoding

`reusable` and `pending_free` must remain logically distinct on disk:

- encode all `reusable_range` records in one canonical section
- encode all pending groups after that section
- do not flatten pending groups into reusable pages during serialization
- preserve empty-state meaning explicitly: zero reusable ranges and zero pending
  groups is a valid freelist image

The decoder should rebuild:

- exact reusable range count
- exact pending group count
- exact per-group `freeing_txn_id` association

No decode path may reconstruct grouping heuristically from page order alone.

## Multi-Page Freelist

### Chain Shape

If the encoded image does not fit in one page, serialize it across a chain of
`PAGE_TYPE_FREELIST` pages:

- metadata references the first freelist page only
- each freelist page stores `freelist_sequence_index`
- each page stores `freelist_page_count`
- each page stores `next_freelist_page_id`, with `0` on the final page
- all pages in the chain are copy-on-write pages with normal checksums

Validation rules for the chain:

- first page must have `freelist_sequence_index = 0`
- sequence indexes must increase by one
- every page in the chain must report the same `freelist_page_count`
- non-final pages must have non-zero `next_freelist_page_id`
- final page must have `next_freelist_page_id = 0`
- traversed page count must equal `freelist_page_count`
- every page ID must be `>= 3` and `< selected_metadata.page_count`
- no page ID may repeat inside the chain

### Stable Sizing Loop

Freelist serialization must handle self-sizing deterministically:

1. assemble the candidate logical image from base freelist state, promotions,
   allocations, and new frees
2. estimate how many freelist pages the image needs
3. allocate that many pages from the allocator view, excluding same-transaction
   frees
4. apply self-accounting rules that remove those allocated freelist pages from
   free sets
5. re-encode the image against the chosen page set
6. if the encoded page count changed, repeat until page count stabilizes

The loop must terminate by making the encoded freelist image depend only on the
final allocated freelist page set, not on transient estimates.

If the implementation wants an upper bound, it may reserve extra freelist pages
and shrink later, but the committed image must still be canonical and must not
list unused reserved pages as free.

## Self-Accounting Rules

Freelist replacement is itself a free-space mutation and must account for both
old and new freelist pages.

### New Freelist Pages

Newly allocated freelist pages:

- are allocated from promoted `reusable` pages first, then tail pages
- must be removed from the candidate `reusable` set before encoding
- must not appear in any pending group
- must not appear in the reusable section

### Old Freelist Pages

The base freelist pages reachable from the writer's pinned metadata snapshot:

- remain visible to older readers until metadata switches
- must be added to `pending_free[current_txn_id]` in the candidate new image
- must not be promoted or reused by the same commit

This means the candidate freelist image must include the soon-to-be-old
freelist chain as a newly freed pending group, even though that same chain is
still the reader-visible freelist before publication.

### No Self-Reference

Validation must reject any candidate image where:

- a new freelist page appears in `reusable`
- a new freelist page appears in any pending group
- a page from the new freelist chain also appears as a referenced old freelist
  page
- a freelist chain references itself through page-body entries

## Commit Integration

### Transaction-Local Assembly

Build the candidate freelist image from these inputs:

- writer base snapshot `reusable`
- base pending groups promoted if the reader cutoff allows them
- base pending groups still blocked by older readers
- pages allocated from reusable removed from the candidate reusable set
- pages newly freed by the transaction appended under `pending_free[new_txn_id]`
- old base freelist pages appended under `pending_free[new_txn_id]`

The writer should perform this assembly before metadata construction, using the
same transaction-local model described by M3 and M6 rather than mutating global
database state in place.

### Commit Phase Placement

This feature plugs into the M4 commit phases as follows:

1. `reserve_tail_pages`
2. `write_data_pages`
3. `sync_data_pages`
4. `build_metadata`
5. `write_metadata`
6. `sync_metadata`
7. `publish_metadata`

Freelist-specific rules:

- finalize the new freelist image before `build_metadata`
- write every freelist page during `write_data_pages`
- include the first new freelist page ID in metadata during `build_metadata`
- never publish the new in-memory freelist root before `publish_metadata`

### Metadata Handoff

The commit path must hand off these final values to metadata construction:

- `root_page_id`
- `freelist_page_id` equal to the first page of the new freelist chain
- `page_count` including any new freelist tail allocations
- `new_generation`
- `new_txn_id`

The metadata page must not be built until the freelist chain length, first page
ID, and final page count are known. The freelist root must be treated exactly
like the tree root: a copy-on-write page reference that becomes authoritative
only when the inactive metadata slot is durably selected.

## Validation

### Decode Validation

When opening or decoding the selected freelist image, validate:

- selected metadata points to `PAGE_TYPE_FREELIST`
- every freelist page checksum is valid
- every chain page has the right page type
- sequence fields and chain length are consistent
- reserved bytes required to be zero are zero
- all page ranges have `page_count > 0`
- all range starts are `>= 3`
- all covered page IDs are `< selected_metadata.page_count`
- reusable ranges are sorted and non-overlapping
- pending groups are sorted by `freeing_txn_id`
- ranges within each pending group are sorted and non-overlapping
- no page appears in more than one reusable range
- no page appears in more than one pending group
- no page appears in both reusable and pending

### Commit-Finalization Validation

Before metadata write, validate the candidate image against the full commit
write set:

- no entry names pages `0`, `1`, or `2`
- no entry names a page `>= candidate page_count`
- no entry names a new branch, leaf, overflow, or freelist page that will be
  live after commit
- no entry overlaps the new freelist chain
- old freelist pages appear only in `pending_free[new_txn_id]`
- the image is canonical after range merging

If validation fails, commit must abort before metadata write and return the
project's normal corruption or invalid-state boundary for internal invariant
failure.

### Checker Reuse

Leave reusable helpers for M7 checker and compaction work:

- freelist chain walker
- decoded logical-image validator
- overlap detector across reusable, pending, and live reachable pages
- pretty-printer or structured findings for duplicate or out-of-range entries

The checker may do deeper reachability cross-checks than `open()`, but both
paths should reuse the same low-level freelist decode and validation rules.

## Crash And Recovery Behavior

This feature must preserve the M4 Class A, B, and C commit model.

### Before Metadata Write

If commit fails or crashes before metadata write begins:

- old metadata remains selected
- old freelist remains authoritative
- partially written new freelist pages are ignored
- any new tail pages remain unreachable and are not inferred as free

### During Metadata Ambiguity

If commit reaches metadata write but metadata sync is not confirmed:

- the current process keeps exposing the old metadata snapshot
- reopen may select either old or new metadata through normal validation
- whichever metadata slot wins determines the authoritative freelist image
- recovery must not merge old and new freelist contents

### After Successful Publication

After metadata sync success and publication:

- new readers use the new freelist root
- old readers continue using the old metadata snapshot they pinned earlier
- the newly pending old freelist pages are not reusable until a later
  transaction proves the reader cutoff is safe

### Restart Promotion

In the M6 single-process model, restart clears the active-reader registry. After
reopen:

- previously persisted pending groups may be considered promotable by the next
  writer
- promotion still becomes durable only through a later committed freelist image
- recovery must not rewrite the freelist during `open()` just because a group is
  now promotable

## Test Plan

### Encoding And Decode Tests

- Encode an empty freelist image and verify decode returns zero reusable ranges
  and zero pending groups.
- Encode mixed reusable ranges and pending groups and verify the decoded image
  preserves exact grouping and canonical order.
- Encode singleton and multi-page ranges and verify merge behavior is
  deterministic.
- Reject malformed records such as zero-length ranges or unsorted group order.

### Multi-Page Freelist Tests

- Serialize a freelist image that fits in one page and verify chain length `1`.
- Serialize a freelist image that requires multiple pages and verify sequence
  fields, chain length, and page ordering.
- Force the sizing loop to grow by one page and verify the final committed image
  is stable and canonical.
- Reject a chain with a cycle, skipped sequence index, wrong page count, or
  non-zero final `next_freelist_page_id`.

### Self-Accounting Tests

- Replace a one-page freelist with a new one-page freelist and verify the old
  freelist page enters `pending_free[current_txn_id]`.
- Replace a multi-page freelist and verify every old freelist page enters the
  pending group exactly once.
- Verify new freelist pages are absent from both reusable and pending sections.
- Verify same-transaction freed tree or overflow pages are not reused for the
  new freelist chain.

### Commit And Metadata Handoff Tests

- Commit a mutation that changes freelist contents and verify metadata points to
  the first page of the new freelist chain.
- Verify `page_count` includes newly allocated freelist tail pages.
- Verify commit failure before metadata write leaves the old freelist selected.
- Verify Class B ambiguity may reopen to old or new freelist, but never a mixed
  image.

### Validation And Corruption Tests

- Corrupt a freelist page checksum and verify the selected metadata slot is
  invalid.
- Corrupt the page type of a selected freelist page and verify `Corruption`.
- Encode duplicate page IDs across reusable and pending and verify decode
  validation fails.
- Encode a new freelist page inside its own reusable section and verify
  commit-finalization validation fails.
- Encode a range that covers page `1`, `2`, or `page_count` and verify
  validation fails.

### Crash And Recovery Tests

- Crash after new freelist pages are written but before metadata write and
  verify the old freelist remains authoritative.
- Crash after metadata write but before metadata sync and verify reopen decides
  through normal slot validation only.
- Crash after metadata sync and verify the new freelist is selected.
- Reopen after a failed commit that extended the physical file and verify ignored
  tail pages are not auto-added to freelist state.

### Reuse And Reader-Safety Tests

- Commit pending frees while an older reader is active and verify they remain
  pending after reopen.
- Close the old reader, start a later writer, and verify eligible groups promote
  to reusable before allocation.
- Verify the allocator reuses promoted pages before growing the file tail.
- Verify a reader pinned before freelist replacement continues using its old
  metadata snapshot safely.

## Acceptance

This feature is accepted when:

- `reusable` and `pending_free` serialize and deserialize exactly, including
  per-group `freeing_txn_id`.
- The freelist image can span multiple `PAGE_TYPE_FREELIST` pages with
  validated sequence and chaining fields.
- Serialization uses a stable sizing loop and produces a canonical image.
- New freelist pages are excluded from both reusable and pending entries.
- Old freelist pages are added to `pending_free[current_txn_id]` during
  replacement.
- Commit writes the new freelist chain before metadata and hands its first page
  ID to metadata construction.
- Validation rejects reserved pages, out-of-range pages, duplicates, overlap,
  malformed ranges, and self-referential freelist images.
- Recovery trusts only the freelist referenced by selected metadata and never
  infers free pages from ignored tail data.
- Class A, B, and C crash behavior for freelist replacement matches the M4
  commit model.
- Restart in single-process mode allows later promotion of persisted pending
  groups without rewriting freelist state during `open()`.
- Tests cover encoding, multi-page serialization, self-accounting, validation,
  metadata handoff, crash behavior, reader safety, and acceptance-level reuse.
