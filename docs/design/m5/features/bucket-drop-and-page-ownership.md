# Bucket Drop And Page Ownership

This document defines the concrete M5 implementation plan for bucket drop and
page ownership. It refines the M5 bucket catalog work in
[`docs/design/m5/m5-implementation-plan.md`](/D:/代码D/zi6db/docs/design/m5/m5-implementation-plan.md)
and stays compatible with the M6 freelist and large-value plan in
[`docs/design/m6/m6-implementation-plan.md`](/D:/代码D/zi6db/docs/design/m6/m6-implementation-plan.md).

M5 must support dropping a bucket without weakening M0-M4 copy-on-write,
metadata-switch, snapshot, or crash-recovery rules. The main requirement is
that bucket removal is both a catalog mutation and a precise reachability
change: the dropped bucket disappears atomically at commit, while every page
previously owned by that bucket is tracked for later safe reuse.

## Goal

M5 is complete for this feature when:

- `dropBucket()` removes the bucket from the writer's staged catalog.
- Every branch and leaf page reachable from the dropped bucket root is
  enumerated exactly once and recorded as pages freed by the dropping
  transaction.
- Existing bucket handles and cursors for that bucket become invalid
  immediately inside the same write transaction.
- Older read transactions keep seeing the pre-drop bucket until they close.
- Commit, rollback, and crash recovery all follow the normal M0/M4 metadata
  selection rules.
- The resulting ownership bookkeeping matches the M6 `pending_free` to
  `reusable` model instead of inventing a separate drop-only mechanism.

## Required Invariants

- Bucket pages are owned by the catalog descriptor visible in the transaction's
  pinned metadata snapshot.
- Dropping a bucket never mutates visible bucket pages in place.
- A dropped page is never placed directly into `reusable`.
- A dropped page is never reused by the same write transaction.
- A dropped page is not reusable while an older reader can still reach the old
  bucket root.
- Recreating a bucket with the same name in the same write transaction allocates
  a new empty root and never exposes the old tree.
- The only committed visibility switch remains publication of the new metadata
  page.

## Scope

This feature covers:

- catalog deletion for an existing bucket descriptor
- traversal of the dropped bucket tree to discover owned pages
- invalidation of same-transaction bucket handles and cursors
- reader visibility rules before and after commit
- handoff into M6-style `pending_free` bookkeeping
- rollback, commit, and crash expectations
- tests and acceptance criteria

This feature does not add:

- nested buckets
- immediate page reuse inside the dropping transaction
- overflow-page freeing logic beyond a required hook for M6 integration
- file compaction or truncation

## Data And Ownership Model

For M5, a bucket owns:

- its root leaf or branch page
- every branch page reachable from that root
- every leaf page reachable from that root

For M6 compatibility, the traversal API must also leave a clear extension point
for future owned overflow chains referenced by leaf values. M5 may not need to
enumerate overflow pages yet if overflow values are not live, but the ownership
model must treat them as part of the bucket once M6 lands.

The ownership boundary is strict:

- catalog pages belong to the catalog tree, not to the dropped user bucket
- pages from other buckets must never be traversed or freed during the drop
- shared ownership is not allowed; if validation finds a malformed tree that
  would cause duplicate reachability, the drop must fail with `Corruption`

## Drop Traversal

### Traversal Contract

The drop path must traverse the bucket tree rooted at the descriptor's
`bucket_root_page_id` and return a complete set of owned page IDs.

Traversal requirements:

- visit every reachable page exactly once
- validate page type and checksum before trusting child pointers
- reject page IDs outside `page_count`
- reject cycles, duplicate visits, or structurally inconsistent child links as
  `Corruption`
- support both single-leaf buckets and multi-level trees
- produce page IDs in a deterministic order for tests and debugging

Recommended implementation shape:

1. Decode the bucket descriptor from the staged catalog snapshot.
2. Run a read-only tree walk from `bucket_root_page_id`.
3. Collect visited page IDs into a transaction-local freed-page accumulator.
4. After successful traversal, remove the catalog entry in the writer's staged
   catalog tree.
5. Append the collected page IDs to `pending_free[current_txn_id]` in the
   writer's staged freelist state.

The traversal must complete before any pages are recorded as freed. If the tree
is corrupt and enumeration is incomplete, `dropBucket()` fails and leaves the
writer state unchanged for that bucket.

### Ordering

The safest ordering for the write transaction is:

1. Look up and validate the catalog entry.
2. Traverse the bucket tree and collect owned pages.
3. Mark the bucket state as dropped in memory so old handles/cursors fail.
4. Clone and update the catalog tree to remove the descriptor.
5. Record the collected pages in `pending_free[current_txn_id]`.

This order avoids publishing a catalog deletion before the writer knows the full
page set that became unreachable.

## Handle Invalidation

`Bucket` handles and `Cursor` instances derived from a bucket being dropped must
be invalidated immediately inside the same write transaction.

Required behavior:

- `getBucket()` for the dropped name returns `null` after the drop unless the
  bucket is recreated later in the same write transaction
- any previously acquired `Bucket` handle for that name fails subsequent
  operations with `InvalidState` or the project's equivalent lifecycle error
- any cursor created from that bucket also fails subsequent operations with
  `InvalidState`
- transaction close still invalidates all remaining handles and cursors in the
  normal way

Recommended implementation:

- assign each bucket handle an in-transaction lifecycle record keyed by bucket
  name plus a local generation or dropped flag
- when `dropBucket()` succeeds, set that record to dropped and increment the
  bucket mutation revision
- cursor operations check both transaction liveness and bucket lifecycle state
  before reading pages

This feature should use the same conservative invalidation model as other M5
write-side cursor mutations instead of trying to keep dropped handles alive
through best-effort rebinding.

## Older Reader Visibility

Reader visibility must stay snapshot-based.

Required behavior:

- a read transaction that started before the writer commits keeps seeing the old
  catalog entry and old bucket contents
- its bucket handles and cursors remain valid until that read transaction closes
- a new read transaction started after commit does not see the dropped bucket
- the dropping write transaction itself must stop seeing the bucket immediately
  after the drop

This means the feature has two simultaneous truths:

- the writer's staged catalog no longer contains the bucket
- older readers can still reach the dropped root through their pinned metadata

That split is the reason dropped pages must enter `pending_free` rather than
`reusable`.

## `pending_free` Semantics

Bucket drop must plug directly into the M6 freelist model.

Required semantics:

- all pages collected by the drop traversal are appended to
  `pending_free[current_txn_id]`
- none of those pages are added to `reusable` during the same transaction
- none of those pages are allocatable by the same transaction, even if the
  bucket is recreated immediately
- later transactions may promote that pending group to `reusable` only when no
  active reader has `snapshot_txn_id < current_txn_id`

M5 should implement ownership recording with M6-compatible terminology now:

- `freed_in_tx` for transaction-local staging, if such a helper exists
- `pending_free[freeing_txn_id]` for the committed grouping
- `reusable` only for pages already proven safe to reallocate

If M5 still has a placeholder freelist implementation, the placeholder must
preserve these semantics even if actual space reuse is deferred to M6. The
critical rule is that drop bookkeeping is exact and durable, not best-effort.

## Rollback, Commit, And Crash Behavior

### Rollback

Rollback discards:

- the staged catalog deletion
- the dropped lifecycle state for writer-local handles and cursors
- the collected freed-page staging for this transaction

After rollback:

- the committed database still contains the bucket
- no pending-free entry from the rolled-back transaction is visible
- older readers and future readers both see the pre-drop state

### Commit

Commit publishes the drop atomically by the normal M0/M4 sequence:

1. write new catalog pages, freelist pages, and any other cloned pages
2. sync non-metadata pages
3. write inactive metadata referencing the new catalog root and freelist root
4. sync metadata
5. switch the in-memory selected metadata

After successful commit:

- new readers do not see the bucket
- the committed freelist contains the dropped pages under
  `pending_free[committing_txn_id]`
- the pages remain unavailable for reuse until reader cutoff rules allow
  promotion

### Crash Behavior

Crash behavior must match the normal metadata-switch model:

- if the crash happens before metadata publication, recovery selects the old
  metadata and the bucket is still present
- if the crash happens after the new metadata is durably selected, recovery sees
  the bucket as dropped and the corresponding pending-free group as committed
- partial drop traversal state or partial freelist images that were never
  selected by metadata are ignored during recovery

The drop path must not require any recovery-time inference such as scanning for
unreachable bucket pages. Recovery trusts only the freelist and catalog
reachable from the selected metadata.

## Implementation Steps

1. Add a bucket-tree traversal helper that walks branch and leaf pages from a
   supplied root and returns a validated page-ID set.
2. Add transaction-local freed-page staging compatible with
   `pending_free[current_txn_id]`.
3. Add bucket lifecycle tracking so dropped handles and cursors fail
   deterministically.
4. Implement `dropBucket()` to perform lookup, traversal, invalidation, catalog
   deletion, and freed-page staging in one write-transaction operation.
5. Ensure `getBucket()` and cursor creation consult the staged catalog view
   after a drop.
6. Ensure rollback discards staged drop ownership and lifecycle changes.
7. Ensure commit serializes the pending-free state with the catalog mutation.
8. Add corruption-path tests for malformed dropped trees and duplicate
   reachability.
9. Add reopen and crash tests proving that recovery selects either the old or
   new drop state, never a mixed state.

## Tests

### Drop Traversal Tests

- Drop a bucket containing a single leaf page and verify exactly that root page
  enters the transaction's freed-page set.
- Drop a multi-level bucket and verify every reachable branch and leaf page is
  recorded exactly once.
- Drop a bucket whose tree is corrupt and verify `dropBucket()` fails without
  deleting the catalog entry or publishing partial freed-page state.

### Handle And Cursor Tests

- Acquire a bucket handle, drop the bucket in the same write transaction, and
  verify `get`, `put`, `delete`, and `cursor` operations fail on the old handle.
- Acquire a cursor, drop the bucket, and verify `first`, `seek`, `next`,
  `prev`, `key`, and `value` fail with lifecycle invalidation rather than
  returning normal end-of-iteration results.
- Drop and then recreate the same bucket name in one write transaction and
  verify the old handle remains invalid while a fresh handle sees the new empty
  bucket.

### Older Reader Visibility Tests

- Start reader R1, open bucket `a`, start writer W1, drop `a`, commit W1, and
  verify R1 still reads the old bucket contents.
- After R1 closes, start reader R2 and verify `a` is missing.
- Verify cursors opened by R1 before the writer commit continue to iterate the
  old snapshot normally.

### `pending_free` Tests

- Drop a bucket and verify its pages land in `pending_free[current_txn_id]`,
  not `reusable`.
- In the same write transaction, recreate the bucket and verify the allocator
  does not reuse the just-dropped page IDs.
- With an older reader active, commit the drop and verify the pending group does
  not promote early.
- After the older reader closes, verify a later write transaction can promote
  the committed pending group according to M6 rules.

### Rollback, Commit, And Crash Tests

- Drop then rollback and verify the bucket remains visible after reopen.
- Drop then commit and verify the bucket is absent after reopen while the
  committed pending group is present.
- Inject a crash before metadata publication and verify recovery selects the old
  bucket and old freelist.
- Inject a crash after metadata publication and verify recovery selects the
  dropped bucket state and the committed pending-free group.

### M6 Forward-Compatibility Tests

- Add a design-level test expectation that once overflow values exist, dropping
  a bucket with overflow-backed values frees both tree pages and overflow pages.
- Verify the traversal interface has a documented extension point for leaf-owned
  overflow references rather than hard-coding leaf-only ownership forever.

## Acceptance

This feature is accepted when all of the following are true:

- `dropBucket()` removes an existing bucket from the writer's staged catalog and
  returns `false` without side effects for a missing bucket.
- The implementation traverses the dropped bucket tree and records every owned
  branch and leaf page exactly once.
- Same-transaction bucket handles and cursors for the dropped bucket are
  invalidated immediately and fail clearly.
- Older read transactions keep seeing the dropped bucket until they close, while
  readers opened after commit do not.
- Dropped pages enter `pending_free[freeing_txn_id]` and are not directly
  reusable.
- The dropping write transaction cannot reuse its own dropped pages, including
  the drop-and-recreate case.
- Rollback restores the pre-drop visible state and discards staged ownership
  bookkeeping.
- Commit publishes catalog deletion and pending-free bookkeeping atomically via
  the normal metadata switch.
- Crash recovery selects either the full pre-drop state or the full post-drop
  state, never a partial mix.
- The test matrix covers traversal, invalidation, snapshot visibility,
  pending-free semantics, rollback, commit, crash, and M6 overflow handoff.
