# M6 Implementation Plan: Space Reuse And Large Values

This document is based on `M6: Space Reuse And Large Values` in
[MILESTONES.md](../../../MILESTONES.md), the product goals in
[ROADMAP.md](../../../ROADMAP.md), and the M0 design freeze under
[`docs/design/m0/`](../m0/).

M6 turns the format hooks reserved in M0 into production behavior:

- overflow pages for values that do not fit efficiently inside one leaf page
- a complete freelist implementation with safe page reuse
- precise `pending_free` to `reusable` transitions
- database statistics and free-space reporting
- tests that prove deleted and replaced space is reused without weakening
  crash safety or reader snapshot safety

M6 must not change M0's on-disk invariants. Metadata remains the only commit
switch point, all visible pages remain copy-on-write, and recovery must only
trust the freelist reachable from the selected metadata.

## 1. Goals

M6 is complete when long-lived databases can grow, delete, update, and continue
writing without leaking old pages indefinitely.

Primary goals:

- Store and read large values through `PAGE_TYPE_OVERFLOW` pages.
- Reclaim pages made unreachable by deletes, updates, bucket drops, and overflow
  replacement.
- Reuse reclaimed pages only when no active reader can still reach them.
- Persist freelist state, including `pending_free`, through the same commit
  protocol as tree pages.
- Expose stable statistics for file size, allocated pages, reusable pages,
  pending pages, overflow pages, and estimated reclaimable bytes.
- Add crash, recovery, snapshot, and corruption coverage for overflow and
  freelist behavior.

## 2. Dependencies And Assumptions

M6 assumes M1-M5 have already delivered:

- single-file open/create/close with fixed page size and validated metadata
- B+Tree leaf and branch page encoding for ordered byte keys and values
- transaction lifecycle with multiple readers and one writer
- crash-safe metadata switching and recovery
- buckets and cursors, including delete and update paths

M6 must integrate with those components rather than redesign them. If earlier
milestones left placeholder freelist or overflow behavior, M6 replaces only the
placeholder implementation while preserving public API compatibility and M0
format compatibility.

M6 should leave M7 concerns as integration points, not blockers:

- cross-process reader lock tables remain an M7 hardening task
- compaction or vacuum remains optional and out of scope for M6
- backup/export remains out of scope for M6

## 3. M0 Constraints That Must Hold

M6 implementation must obey these frozen M0 decisions:

- Page IDs are unsigned 64-bit logical identifiers.
- Page size is fixed per database and defaults to `4096`.
- Pages `0`, `1`, and `2` are reserved for the header and metadata slots.
- Normal pages use a common header with `page_id`, `page_type`,
  `page_generation`, flags, and checksum.
- `PAGE_TYPE_OVERFLOW` is the only page type for spilled values.
- The freelist page stores both `reusable` page IDs and
  `pending_free[freeing_txn_id]` groups.
- Deleted pages first enter `pending_free[current_txn_id]`.
- A pending page may move to `reusable` only after all readers older than the
  freeing transaction have ended.
- Recovery trusts only the freelist page referenced by selected metadata.
- Recovery must not reclaim orphaned tail pages or infer free pages from
  unreachable data pages.
- Commit writes all non-metadata pages, syncs them, writes inactive metadata,
  syncs metadata, then switches in-memory metadata.

## 4. Overflow Page Implementation

### 4.1 Value Placement Policy

Each leaf value is stored in one of two forms:

- `inline`: value bytes are stored directly in the leaf payload area
- `overflow`: leaf stores a small overflow reference and the value bytes are
  stored across one or more overflow pages

The storage engine should choose overflow when either condition is true:

- the value cannot fit in the target leaf after accounting for key, slot, and
  split overhead
- storing the value inline would make leaf split behavior pathological for
  normal workloads

Recommended initial threshold:

- use inline storage when the full key/value entry can fit within a normal leaf
  target occupancy after a split
- use overflow for values larger than `page_size / 4`, unless the M2/M5 page
  builder already computes a stricter fit threshold

The threshold is an implementation policy, not a file-format rule. Once a value
is written, reads must follow the encoding stored in the leaf slot.

### 4.2 Leaf Overflow Reference

M0 reserved value flags for overflow references without changing the leaf slot
layout. M6 should bind that reservation as follows:

- a leaf slot with `VALUE_FLAG_OVERFLOW` set does not interpret the value payload
  bytes as the user value
- `value_offset` and `value_length` point to an overflow-reference record inside
  the leaf payload area
- the overflow-reference record contains at least:
  - `first_overflow_page_id`
  - `overflow_page_count`
  - `logical_value_length`
  - reserved flags or checksum extension space

This keeps the leaf slot shape stable. The overflow-reference record is part of
the leaf page body and is protected by the leaf checksum.

If earlier milestones bound `value_offset` or `value_length` differently, M6
must adapt inside the already reserved value payload and flags. It must not add
new required fields to the common leaf slot without a format-version review.

### 4.3 Overflow Page Body

Each overflow page uses the M0 common page header with `page_type = OVERFLOW`.
The page body should contain:

- `owner_txn_id` or `page_generation` inherited from the common page generation
- `chain_id` or first-page ID used for validation and diagnostics
- `index_in_chain`
- `next_overflow_page_id`, where `0` means end of chain
- `logical_value_length` on the first page
- `payload_length` for this page
- payload bytes
- reserved bytes must be zeroed before checksum calculation

Required validation:

- page header `page_id` matches the physical page being read
- page type is `OVERFLOW`
- checksum validates before payload use
- `first_overflow_page_id` in the leaf reference equals the first page read
- `overflow_page_count` is non-zero when `VALUE_FLAG_OVERFLOW` is set
- `index_in_chain` is monotonic from `0`
- every page's `chain_id` or first-page field matches the first page ID
- the final page has `next_overflow_page_id = 0`
- non-final pages have a non-zero next page ID
- chain length equals `overflow_page_count` from the leaf reference
- the sum of payload lengths equals `logical_value_length`
- no overflow chain page appears twice in the same chain
- all page IDs are `< selected_metadata.page_count`
- each page's payload length is within the usable payload capacity of an
  overflow page

### 4.4 Overflow Allocation

M6 should allocate overflow pages through the same transaction page allocator as
branch, leaf, and freelist pages.

Allocation rules:

- allocate from `reusable` first, then from the file tail
- never allocate from pages freed by the same transaction; only a later
  transaction may reuse them after the reader safety rule is satisfied
- track all pages allocated for a value as owned by the active write transaction
- write overflow pages before metadata, under the same checksum and sync rules
  as B+Tree pages
- remove every reused page from the transaction-local `reusable` set at
  allocation time so the same page ID cannot be allocated twice
- keep newly allocated overflow pages out of `pending_free` unless the
  transaction later abandons that value before commit

The first implementation may prefer contiguous tail allocation for new chains
when available, but correctness must not depend on contiguity. A chain may span
arbitrary page IDs.

The overflow writer should build the complete chain in memory before exposing
the leaf reference in a cloned leaf page. If allocating or encoding any page in
the chain fails, the write operation must discard the partial chain from the
transaction's private state and leave the logical key unchanged.

### 4.5 Reading Large Values

Reads must be snapshot-stable:

- a read transaction follows only the root and freelist snapshot selected when
  it began
- an overflow chain referenced by that snapshot must remain readable until the
  read transaction closes
- replacement or deletion by a later write transaction must not mutate or free
  the old chain into `reusable` while older readers are active

Initial API behavior can return a copied buffer for overflow values even if
small inline values use lower-copy access. A future zero-copy API is not required
for M6.

Overflow validation should run when an overflow value is read. `open()` is only
required to validate the metadata-selected root and freelist structures from M0;
full traversal of every live overflow chain may remain an internal checker or
M7 verify responsibility. A read that encounters a malformed live overflow chain
must fail with `Corruption` rather than returning partial bytes.

### 4.6 Updating And Deleting Large Values

Replacing an overflow value must:

1. allocate and write a new inline value or new overflow chain
2. update the cloned leaf slot to point to the new value representation
3. append the old overflow chain pages to `pending_free[current_txn_id]`
4. append old cloned/replaced tree pages to `pending_free[current_txn_id]`
5. commit the new root, new freelist page, and metadata atomically

Deleting an overflow value must:

1. clone the affected leaf path
2. remove the leaf entry
3. validate enough of the old overflow reference to enumerate the exact chain
4. append every page in the old overflow chain to
   `pending_free[current_txn_id]`
5. commit the new tree and freelist state through the normal M0 commit protocol

If the old overflow chain cannot be enumerated because it is corrupt, the
delete or replacement must fail with `Corruption`. It must not drop the leaf
reference and leak or guess pages to free.

Rollback discards only in-memory references to the newly allocated chain and
pending-free updates. Any tail pages written before rollback are ignored unless
future committed metadata references them.

## 5. Freelist And Reuse Policy

### 5.1 Freelist Data Model

The in-memory freelist should mirror the persisted freelist:

- `reusable`: page IDs immediately available to a write transaction
- `pending_free`: map from `freeing_txn_id` to page ID ranges or page ID lists
- `allocated_in_tx`: page IDs allocated by the current write transaction
- `freed_in_tx`: page IDs newly freed by the current write transaction

The persisted freelist page must include enough information to rebuild
`reusable` and `pending_free` exactly during `open()`. The engine must not scan
the data file and infer free pages during recovery.

### 5.2 Pending And Reusable Semantics

When a write transaction removes the final reference to a page, the page enters:

```text
visible clean -> pending_free[freeing_txn_id]
```

It may later transition to:

```text
pending_free[freeing_txn_id] -> reusable
```

only when both conditions hold:

- no active reader has `snapshot_txn_id < freeing_txn_id`
- the pending group is not the current writer's own
  `pending_free[current_txn_id]`

Equivalently, if an `oldest_active_reader_txn_id` exists, a pending group with
`freeing_txn_id = X` is reusable when `oldest_active_reader_txn_id >= X`. The
equality case is safe because a reader at transaction `X` started after the
delete became visible and cannot reach the pre-delete pages through its pinned
root. Readers with transaction IDs lower than `X` may still reach those pages.

For the single-process implementation before M7:

- active reader transaction IDs are tracked in process memory
- after restart, there are no surviving in-process readers
- eligible persisted pending groups can be treated as promotable after recovery
- any persisted change from pending to reusable still happens by writing a new
  freelist image in a normal write commit

For the future M7 multi-process implementation:

- the transition must use a reader lock table or equivalent durable reader
  visibility mechanism
- M6 should expose an internal boundary where `oldest_active_reader_txn_id` is
  computed, so M7 can replace the single-process implementation

### 5.3 Allocation Order

The allocator should use this order:

1. Move eligible pending groups into `reusable`.
2. Allocate exact page-count runs from reusable ranges when possible.
3. Allocate individual pages from reusable if a contiguous run is unavailable.
4. Allocate from the file tail by increasing transaction-local `page_count`.

Correctness must not require contiguous reuse. Contiguous allocation is a
performance optimization for overflow chains.

Allocator invariants:

- a page ID may appear in at most one of `reusable`, `pending_free`,
  `allocated_in_tx`, and `freed_in_tx`
- allocation from `reusable` removes the page ID or range before returning it
- tail allocation only returns IDs greater than or equal to the transaction's
  starting `page_count` and then advances the transaction-local `page_count`
- fixed pages `0`, `1`, and `2` are never allocatable or freeable
- a page ID reachable from the writer's base root or base freelist must never be
  allocated unless it first became reusable under the reader cutoff rule in a
  prior transaction

### 5.4 Same-Transaction Reuse Rule

Pages freed by the active writer must not be reused by that same transaction.
They remain in `pending_free[current_txn_id]` until a later transaction proves no
older reader can still reach them.

This conservative rule avoids subtle cases where:

- a read transaction still follows the old root
- a write transaction deletes an old overflow chain
- the same write transaction reuses a page ID for a new chain
- the old reader would then observe unrelated bytes through its pinned snapshot

### 5.5 Reuse Of Tree Pages And Overflow Pages

The freelist does not distinguish page types for allocation. A page freed from a
leaf, branch, freelist page, or overflow chain may later be reused for any normal
page type once it is in `reusable`.

Before reuse:

- overwrite the full target page body
- set the new `page_type`
- set the new `page_generation`
- zero reserved fields
- recompute the checksum

No old page-type data may leak into validation-visible reserved fields.

### 5.6 Freelist Page Growth

M6 must handle freelist content that does not fit in one page.

Preferred initial approach:

- support a chain of `PAGE_TYPE_FREELIST` pages using reserved body space
- metadata points to the first freelist page
- freelist pages are copy-on-write and checksummed like other normal pages
- every freelist page includes sequence information and a `next_freelist_page_id`

If earlier milestones only support one freelist page, M6 must either implement
freelist-page chaining or define a bounded error such as `FreelistFull` before
large-scale space reuse is considered complete. The preferred M6 exit path is
chaining, because long-lived databases can accumulate many ranges and pending
groups.

Freelist encoding should prefer sorted ranges over raw page ID lists:

- ranges reduce space when large overflow chains or tail regions are freed
- raw singleton entries can still be represented as one-page ranges
- pending groups remain keyed by `freeing_txn_id`

### 5.7 Persisted Freelist Commit Rules

The write transaction builds a new freelist image from:

- base `reusable` from its starting snapshot
- eligible pending groups promoted before allocation
- pages allocated from `reusable` removed from that set
- newly freed pages added to `pending_free[current_txn_id]`
- new freelist pages allocated for the freelist image itself

The new freelist page or chain is written as non-metadata data before metadata.
The inactive metadata slot then references the new freelist root page.

If commit fails before metadata publication, the old metadata continues to point
to the old freelist and the new freelist image is ignored by recovery.

Freelist self-accounting rules:

- old freelist pages from the writer's base snapshot are freed into
  `pending_free[current_txn_id]` when replaced by a new freelist image
- newly allocated freelist pages are not listed in `reusable` or
  `pending_free` in the freelist image they store
- if serializing the freelist requires more pages than initially allocated, the
  writer must allocate additional freelist pages and rebuild the image until the
  encoded page count is stable
- a committed freelist image must be internally sorted or canonicalized so
  duplicate detection is deterministic during recovery and tests
- metadata publication is the only point at which the new freelist root becomes
  authoritative

The writer must validate the candidate freelist image before writing metadata.
Validation must reject fixed page IDs, out-of-range page IDs, duplicate entries,
overlap between reusable and pending sets, and entries that name pages allocated
for the new root, tree pages, overflow pages, or the new freelist image.

### 5.8 Space Reuse Safety Boundaries

Space reuse must be tied to transaction boundaries, not background activity.

Required boundaries:

- pending promotion may run at `beginWrite()` or during commit finalization, but
  the result becomes durable only through the new freelist page and metadata
  commit
- read transactions keep using the page IDs reachable from their pinned metadata
  even if a later writer has moved those page IDs into pending
- no page ID may be returned by the allocator after it is freed until a later
  transaction observes a safe reader cutoff
- a failed or rolled-back writer must not publish its allocation removals,
  pending additions, or pending promotions
- recovery after an ambiguous commit must use only the freelist referenced by
  the metadata slot selected during M4 recovery

These boundaries apply equally to branch, leaf, overflow, and freelist pages.
M6 must not add special reuse paths for overflow pages that bypass the central
freelist.

## 6. Large Value Semantics

### 6.1 Maximum Value Size

M6 should define an explicit maximum value size instead of relying on arithmetic
overflow checks scattered through the code.

Recommended public rule:

- keys and values remain raw byte slices
- maximum value length is bounded by addressable page count, page size, and
  allocator limits
- attempts to write a value larger than the supported limit return a clear
  error before modifying transaction state

Recommended internal checks:

- `logical_value_length` must fit in `u64`
- `overflow_page_count * usable_payload_per_page` must not overflow
- total allocation must fit below the metadata `page_count` limit
- memory allocation for readback must respect the allocator's maximum size

### 6.2 Empty Values And Boundary Values

M6 must preserve existing small-value behavior:

- empty values remain legal inline values
- boundary values around the overflow threshold must round-trip exactly
- replacing an overflow value with an inline value frees the old overflow chain
- replacing an inline value with an overflow value frees only the old cloned leaf
  pages, not unrelated pages

### 6.3 Cursor Behavior

Cursors from M5 must expose the same logical values regardless of storage form.

Required behavior:

- `cursor.value()` returns the value bytes for inline and overflow values
- cursor movement must not materialize unrelated overflow values
- if the implementation returns borrowed slices for inline values, overflow
  values may return transaction-owned scratch buffers or an explicit read API
  result type, as long as lifetime rules are documented
- closing or rolling back the transaction invalidates any overflow buffers tied
  to that transaction

## 7. Statistics API

### 7.1 API Shape

M6 should add a small read-only statistics API. Names may follow the final Zig
style used by earlier milestones, but the data model should include:

```zig
pub const DBStats = struct {
    page_size: u32,
    file_size_bytes: u64,
    page_count: u64,
    allocated_pages: u64,
    metadata_pages: u64,
    branch_pages: u64,
    leaf_pages: u64,
    overflow_pages: u64,
    freelist_pages: u64,
    reusable_pages: u64,
    pending_pages: u64,
    free_pages_total: u64,
    used_pages_estimate: u64,
    reusable_bytes: u64,
    pending_bytes: u64,
    free_bytes_total: u64,
    largest_reusable_run_pages: u64,
    active_read_transactions: u64,
    oldest_reader_txn_id: ?u64,
};

pub fn stats(db: *DB) !DBStats;
pub fn statsTx(tx: *const Tx) !DBStats;
```

`stats(db)` reports the current selected in-memory metadata snapshot.
`statsTx(tx)` reports the transaction's pinned snapshot. This distinction is
important because read transactions may be older than the current database
state.

### 7.2 Stats Consistency

Stats do not need to run a full integrity check. They should be fast and based
on metadata plus freelist accounting, with optional page-type counters if the
implementation already maintains them.

Required consistency:

- `file_size_bytes = page_count * page_size` for logical reporting from the
  selected metadata snapshot
- `free_pages_total = reusable_pages + pending_pages`
- `free_bytes_total = free_pages_total * page_size`
- `reusable_bytes = reusable_pages * page_size`
- `pending_bytes = pending_pages * page_size`
- stats from a read transaction remain stable for that transaction
- stats after a committed write reflect the newly selected metadata
- pending promotions are reflected only in the snapshot whose freelist image
  contains that promotion
- failed commits and ignored tail pages do not change logical stats

If exact branch/leaf/overflow counters require a tree traversal, the API should
either:

- compute them in a slower explicit `collectStats` path, or
- mark them as estimates derived from maintained counters

Do not silently mix exact and approximate semantics without field names or docs
that make the distinction clear.

M6 should distinguish logical size from physical file bytes when the platform
reports a larger file after failed tail growth or ambiguous commits:

- `file_size_bytes` means the logical size implied by selected metadata
- an optional diagnostic such as `physical_file_size_bytes` may report the OS
  file length if the project wants to expose ignored tail pages
- ignored tail pages are not counted as reusable, pending, allocated, or used
  until a future committed metadata snapshot includes them in `page_count`

Counter accuracy rules:

- `reusable_pages` and `pending_pages` must be exact counts decoded from the
  selected freelist image
- `overflow_pages` must be exact only if maintained transactionally or computed
  by a traversal; otherwise the field must be named as an estimate
- `allocated_pages` should mean `page_count`, including fixed pages, unless the
  implementation chooses a different name and documents it clearly
- `used_pages_estimate = page_count - free_pages_total` must never subtract
  ignored physical tail pages outside selected metadata
- stats must not mutate freelist promotion state in read-only transactions

### 7.3 Space Reporting Terms

Use stable terms in docs, tests, and API comments:

- `page_count`: exclusive upper bound of allocated page IDs from selected
  metadata
- `file_size_bytes`: logical file size implied by `page_count`
- `reusable_pages`: free pages available to the next writer
- `pending_pages`: free pages waiting for older readers to close
- `free_pages_total`: reusable plus pending pages
- `used_pages_estimate`: `page_count - free_pages_total`, including fixed
  header and metadata pages
- `largest_reusable_run_pages`: largest contiguous reusable range, useful for
  diagnosing fragmentation

`pending_pages` are free in the logical sense but not available for immediate
reuse. User-facing docs must not describe them as "available" space.

## 8. Execution Order

### Phase 1: Audit Existing Placeholders

Deliverables:

- identify current value encoding and leaf flag reservations
- identify current allocator and freelist placeholder behavior
- identify transaction reader tracking and oldest-reader calculation
- document any M6 blockers found in M1-M5 code

Exit criteria:

- no storage-format changes are needed beyond binding M0-reserved fields
- M6 can be implemented behind existing transaction and page abstractions

### Phase 2: Freelist Core

Deliverables:

- implement in-memory reusable ranges
- implement persisted `pending_free[freeing_txn_id]`
- implement promotion based on `oldest_active_reader_txn_id`
- implement allocation from reusable before tail growth
- implement freelist serialization/deserialization

Exit criteria:

- deletes place pages in pending groups
- eligible pending groups promote to reusable
- later writes reuse pages before growing the file
- recovery reloads freelist state only from selected metadata

### Phase 3: Freelist Commit Integration

Deliverables:

- new freelist image participates in copy-on-write commit
- freelist pages are checksummed and validated
- commit failure leaves old freelist authoritative
- restart migration of eligible pending groups is safe in single-process mode

Exit criteria:

- M0 crash matrix still passes
- unreachable tail pages from failed commits are not auto-added to freelist
- old readers prevent reuse until they close

### Phase 4: Overflow Page Encoding

Deliverables:

- bind `VALUE_FLAG_OVERFLOW`
- encode/decode overflow-reference records in leaf payloads
- write and validate overflow chains
- read large values through transaction snapshots

Exit criteria:

- large values round-trip across reopen
- corruption in an overflow page returns `Corruption` or the internal validation
  error mapped by the existing error model
- inline values still use the existing fast path

### Phase 5: Update/Delete Integration For Large Values

Deliverables:

- replacement of overflow with overflow
- replacement of overflow with inline
- replacement of inline with overflow
- deletion of overflow values
- bucket drop or subtree deletion frees reachable overflow chains

Exit criteria:

- old overflow chains enter pending groups
- reusable pages are consumed by later writes
- read transactions pinned before replacement keep seeing the old value

### Phase 6: Statistics And Space Reporting

Deliverables:

- `DBStats` or equivalent public API
- transaction-pinned stats path if the transaction API supports it cleanly
- documentation comments for pending vs reusable space
- tests for stats before and after commits, deletes, reader closure, and reopen

Exit criteria:

- users can distinguish file size, used pages, reusable pages, and pending pages
- stats are stable within a read transaction

### Phase 7: Fault Injection And Regression Coverage

Deliverables:

- overflow-specific crash tests
- freelist-specific crash tests
- corruption tests for overflow and freelist pages
- regression tests for boundary value sizes and space reuse

Exit criteria:

- M0 crash and corruption expectations still pass
- M6-specific tests prove both correctness and reuse

### Phase 8: M7 Handoff Seams

Deliverables:

- reusable validation helpers for overflow chains and freelist images
- checker findings that distinguish corruption from informational ignored tail
  pages
- a stable internal `oldest_active_reader_txn_id` provider that M7 can replace
  with cross-process reader lock state
- stats fields and terminology that M7 compaction can reuse to report before
  and after space movement

Exit criteria:

- M7 verify can traverse M6 overflow chains without duplicating encoders
- M7 compaction can consume freelist and live-page accounting without changing
  M6's on-disk semantics
- M6 does not perform file truncation, page relocation, or vacuum-style rewrite
  as part of normal delete/update paths

## 9. Test Matrix

### 9.1 Overflow Round-Trip Tests

| Case | Operation | Expected Result |
| --- | --- | --- |
| inline boundary minus one | put/get/reopen | value remains inline and round-trips |
| inline boundary plus one | put/get/reopen | value uses overflow and round-trips |
| one overflow page | put/get/reopen | chain length is `1` |
| multiple overflow pages | put/get/reopen | chain length matches payload size |
| empty value | put/get/reopen | remains empty inline value |
| many large values | insert/reopen/scan | all keys and values remain ordered |

### 9.2 Large Value Update Tests

| Case | Operation | Expected Result |
| --- | --- | --- |
| inline to overflow | update existing key | old inline value disappears, new value reads back |
| overflow to inline | update existing key | old overflow pages enter pending |
| overflow to overflow smaller | update existing key | old chain pending, new chain valid |
| overflow to overflow larger | update existing key | allocator reuses eligible pages then grows tail |
| delete overflow | delete key | old chain pending, key missing after commit |
| rollback overflow put | put then rollback | value missing, no committed freelist change |

### 9.3 Reader Safety Tests

| Case | Setup | Expected Result |
| --- | --- | --- |
| old reader pins overflow | read tx starts, writer replaces value | old reader sees old value |
| pending blocked by reader | old reader active, writer deletes value | freed pages remain pending |
| pending promotes after reader closes | close old reader, start writer | pages become reusable |
| same transaction delete and put | writer deletes then writes large value | pages freed in same tx are not reused |
| new reader after commit | writer commits replacement, new read tx starts | new reader sees new value |

### 9.4 Freelist Reuse Tests

| Case | Operation | Expected Result |
| --- | --- | --- |
| delete then write after reader closes | free pages then insert | file does not grow if reusable pages suffice |
| update large value repeatedly | many replacements | page count stabilizes after reusable pool forms |
| bucket drop with large values | drop bucket/table | all reachable pages become pending |
| restart with pending groups | close/reopen single-process DB | eligible pending groups can promote |
| fragmented reusable pages | free non-contiguous pages | allocator can reuse non-contiguous pages |
| contiguous reusable run | free overflow chain | largest reusable run is reported |
| reused page type changes | free overflow, reuse as leaf | old reserved fields are zeroed and checksum validates |
| old freelist replacement | commit with new freelist chain | old freelist pages enter pending, new freelist pages are not free |
| duplicate allocation guard | allocate from reusable ranges | same page ID is not returned twice |

### 9.5 Crash And Recovery Tests

| Fault Point | Expected Recovery |
| --- | --- |
| crash before overflow page writes | old metadata selected, old value visible |
| crash during overflow page writes | old metadata selected, partial chain ignored |
| crash after overflow pages sync before metadata | old metadata selected |
| crash during metadata write for large value commit | newest valid metadata selected |
| crash after metadata sync for large value commit | new metadata selected |
| crash after freelist page write before metadata | old freelist remains authoritative |
| crash after metadata points to new freelist | new freelist validates and is authoritative |
| ENOSPC during overflow tail allocation | commit fails before metadata, old state selected |
| short write of freelist page | commit fails or metadata invalid, old state selected |
| crash after pending promotion before metadata | old freelist remains authoritative |
| crash after reuse page write before metadata | old metadata selected, reused page remains invisible |
| metadata selects reused pages after sync | new metadata selected, reused page type/checksum valid |

### 9.6 Corruption Tests

| Corruption | Expected Result |
| --- | --- |
| overflow page wrong type | read/open validation reports corruption |
| overflow checksum mismatch | read/open validation reports corruption |
| overflow chain cycle | read reports corruption |
| overflow chain shorter than reference | read reports corruption |
| overflow chain longer than reference | read reports corruption |
| overflow page ID out of range | read/open validation reports corruption |
| overflow final page has non-zero next | read reports corruption |
| overflow page shared by two live values | checker reports corruption |
| delete sees corrupt overflow chain | delete fails and does not guess freed pages |
| freelist page checksum mismatch | open reports corruption if selected |
| freelist references metadata pages | open reports corruption |
| freelist references duplicate page IDs | open reports corruption |
| page appears in both reusable and pending | open reports corruption |
| freelist references new freelist page itself | open/checker reports corruption |

### 9.7 Stats Tests

| Case | Expected Result |
| --- | --- |
| empty database | fixed pages and empty freelist are reported clearly |
| after large insert | overflow_pages and file_size increase |
| after large delete with reader active | pending_pages increases, reusable_pages unchanged |
| after reader closes | reusable_pages increases after promotion point |
| after reuse | reusable_pages decreases and page_count does not grow unnecessarily |
| transaction-pinned stats | old read tx stats remain stable across writer commit |
| reopen stats | freelist and page counts survive recovery |
| ignored physical tail pages | logical file_size_bytes follows metadata page_count |
| pending promotion commit | reusable_pages changes only after committed freelist metadata |

### 9.8 M7 Handoff Tests

| Case | Expected Result |
| --- | --- |
| checker traversal of overflow values | live overflow chains are enumerated once |
| checker traversal with ignored tail pages | ignored tail pages are informational, not reusable |
| compaction preflight accounting | live pages plus freelist pages are classified without overlap |
| reader cutoff provider swap | single-process provider can be replaced without allocator changes |

## 10. Validation Rules

M6 should add internal validation helpers that can be reused by M7 verify tooling.

Required checks:

- metadata `freelist_page_id` points to valid freelist page or chain
- freelist pages are checksummed and have the right page type
- freelist entries never include pages `0`, `1`, or `2`
- freelist entries are `< page_count`
- no duplicate page appears in reusable ranges
- no duplicate page appears across pending groups
- no page appears in both reusable and pending
- active tree traversal and freelist accounting do not claim the same page
- overflow references point to valid chains
- overflow chain pages are not shared by two live values

Some checks may be too expensive for every `open()`. At minimum, `open()` must
validate critical metadata and selected freelist structure as required by M0.
Deeper reachability validation can be placed behind an internal checker that M7
later exposes as a verify API.

## 11. Error Handling

Use the existing M0 error model where possible:

- unsupported format changes -> `IncompatibleVersion`
- malformed overflow/freelist structure -> `Corruption`
- checksum mismatch -> internal `ChecksumMismatch` or public `Corruption`
- transaction misuse after commit/rollback/failure -> `TransactionClosed`
- value too large for supported limits -> add a specific error only if the
  project already allows extending the error set; otherwise use `InvalidState`
  with a clear message or internal diagnostic
- allocation failure, ENOSPC, short write, or sync failure -> `IoError`

Commit failure must leave the write transaction terminal and must not switch the
current in-memory metadata unless the M0 success point is reached.

## 12. Performance Notes

M6 should prioritize correctness and clear accounting over advanced tuning.

Allowed optimizations:

- sorted ranges for freelist storage
- best-fit or first-fit reusable range allocation
- contiguous tail allocation for new overflow chains
- cached freelist counts for stats
- streaming overflow reads when the public API supports it

Avoid:

- background reclamation threads
- asynchronous page reuse that is not tied to transaction boundaries
- file truncation as part of normal delete behavior
- scanning the whole file on startup to rebuild freelist state

## 13. M7 Verify And Compaction Handoff

M6 should leave the database easier to verify and compact in M7, but it must not
silently implement partial compaction semantics.

Verify handoff:

- expose internal read-only walkers for the selected metadata root, freelist
  page chain, and overflow chains
- report duplicate ownership separately for tree reachability, overflow chain
  sharing, reusable freelist entries, and pending freelist entries
- classify ignored tail pages outside selected `page_count` as informational
  unless they are referenced by selected metadata
- return structured findings that M7 can map to a public verify API or CLI

Compaction handoff:

- keep normal deletes and updates as page reuse only; they must not truncate or
  rewrite the file tail
- maintain enough live-page accounting for M7 to copy live tree, overflow, and
  bucket pages into a new compacted layout
- ensure freelist encoding can represent large free ranges produced by a future
  compaction or vacuum
- do not expose user-facing promises that deleted space will reduce physical
  file size before M7 implements compaction

The M6 exit promise is safe reuse inside the existing file. Physical shrinking,
page relocation, backup/export, and public verify tooling remain M7 scope.

## 14. Non-Goals

M6 does not implement:

- SQL or secondary indexes
- multi-writer commits
- cross-process reader lock table finalization
- compaction, vacuum, or file shrinking
- backup or snapshot export
- encryption or compression of overflow values
- mmap-based storage
- best-effort repair of corrupted overflow or freelist pages
- changing the page size, page type constants, checksum algorithm, or metadata
  commit protocol

## 15. Acceptance Checklist

M6 can exit when all of the following are true:

- Large inline-boundary and multi-page values can be written, read, scanned, and
  reopened correctly.
- Overflow pages use `PAGE_TYPE_OVERFLOW` and are protected by normal page
  checksums.
- Overflow references are encoded through M0-reserved leaf value flags and do
  not require a leaf slot format break.
- Replacing or deleting an overflow value places all old overflow pages in
  `pending_free[current_txn_id]`.
- Deleted tree, freelist, and overflow pages are reused by later writes after
  reader-safety rules permit reuse.
- Pages freed by the active writer are not reused by the same transaction.
- Active older readers prevent pending pages from becoming reusable.
- After older readers close, eligible pending groups can move to reusable.
- Restart in single-process mode can promote eligible pending groups without
  inferring free space from unreachable file pages.
- Commit and recovery behavior still matches the M0 crash matrix.
- Freelist and overflow corruption cases return clear corruption errors.
- Statistics expose file size, page count, reusable pages, pending pages, total
  free pages, and free-space bytes.
- Stats distinguish reusable space from pending space.
- Stats from a read transaction are stable across later writer commits.
- Logical stats ignore physical tail pages that are outside selected metadata.
- Tests cover overflow boundaries, update/delete transitions, reader safety,
  reuse behavior, crash points, corruption cases, and stats reporting.
- Internal validation helpers are ready for M7 verify and compaction preflight
  without changing M6 storage semantics.
- Documentation and API comments use the stable terms `pending_free`,
  `reusable`, `page_count`, `free_pages_total`, and `file_size_bytes`.
