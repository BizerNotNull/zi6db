# M6 Feature Plan: Overflow Allocation, Read, Update, Delete

This plan defines the implementation details for the M6 feature
`overflow allocation read update delete`. It is derived from
[`docs/design/m6/m6-implementation-plan.md`](../m6-implementation-plan.md),
the transaction and snapshot rules in
[`docs/design/m3/m3-implementation-plan.md`](../../m3/m3-implementation-plan.md),
and the repository guidance in `AGENTS.md`.

The scope of this document is limited to the lifecycle of overflow-backed
values:

- allocating overflow chains for large values
- reading overflow values through stable snapshots
- replacing overflow-backed values with inline or new overflow values
- deleting overflow-backed values
- freeing old chains through `pending_free` and later reuse
- defining tests and acceptance criteria for this feature

This feature must preserve all M0 and M3 invariants:

- visible pages are never modified in place
- metadata remains the only commit switch point
- readers use the metadata snapshot pinned at `beginRead()`
- freed pages first enter `pending_free[freeing_txn_id]`
- pages freed by the active writer are never reused by that same transaction

## 1. Feature Goals

This feature is complete when the engine can store, read, replace, and delete
large values without breaking snapshot isolation or page reuse safety.

Required outcomes:

- Large values can spill into `PAGE_TYPE_OVERFLOW` chains.
- Reads follow the overflow reference stored in the leaf snapshot selected by
  the transaction.
- Replacing or deleting a large value moves the old overflow chain into
  `pending_free[current_txn_id]`.
- Rollback discards all new overflow allocations and leaf changes from the
  uncommitted transaction view.
- Later transactions can reuse old overflow pages only after the M3 reader
  cutoff rule says that reuse is safe.

## 2. Allocation Order

Overflow page allocation must use the same allocator and safety rules as other
normal pages.

### 2.1 Allocation sequence

For any write that needs an overflow chain:

1. Promote eligible old `pending_free` groups into `reusable` using the current
   oldest active reader cutoff.
2. Attempt to satisfy the required overflow page count from `reusable`.
3. Prefer a contiguous reusable run when one exists and is large enough.
4. Fall back to multiple reusable pages when no single run is large enough.
5. Grow from the logical file tail only for any remaining pages.

Correctness must not depend on contiguity. Contiguous chains are an
optimization, not a format rule.

### 2.2 Transaction-local allocation rules

The writer must track every allocated overflow page in transaction-local state.

Required invariants:

- allocation removes a reused page from the transaction-local reusable view
  immediately
- one page ID cannot be returned twice in the same transaction
- pages allocated for a new overflow chain are not added to `pending_free`
  unless that chain later becomes the old value in a subsequent committed
  transaction
- pages freed by the current transaction are not reusable by the current
  transaction, even if the writer deletes and reinserts the same key
- tail allocation advances the transaction-local `page_count` only for pages
  that remain part of the candidate commit image

### 2.3 Overflow chain construction

The writer should build the full chain before publishing the new leaf slot in
the cloned tree path.

Implementation steps:

1. Determine whether the new value remains inline or requires overflow storage.
2. Compute the exact overflow page count from value length and usable payload
   per overflow page.
3. Reserve page IDs through the shared allocator.
4. Encode each overflow page with final page headers, chain metadata, payload
   length, and checksum.
5. Encode the leaf overflow-reference record with:
   `first_overflow_page_id`, `overflow_page_count`, and
   `logical_value_length`.
6. Update the cloned leaf entry only after the new chain is fully represented
   in transaction-local state.

If any allocation, encoding, or serialization step fails, the write must abort
without changing the logical key in the committed view.

## 3. Snapshot Reads

Overflow reads must follow the M3 snapshot model exactly.

### 3.1 Snapshot source of truth

A read transaction must use only:

- its pinned root page ID
- its pinned `page_count`
- its pinned freelist page ID when freelist-aware checks are needed
- leaf overflow references reachable from that pinned root

It must never consult:

- the active writer's dirty pages
- newer metadata published after the transaction began
- allocator state derived from later commits

### 3.2 Overflow read flow

When a leaf slot has the overflow flag set, the read path should:

1. Decode the overflow-reference record from the leaf payload.
2. Validate the reference fields before reading any page.
3. Read the first overflow page using the transaction's pinned snapshot rules.
4. Walk the chain page by page until `overflow_page_count` pages are consumed.
5. Validate page ID, page type, checksum, chain identity, chain order, and
   payload lengths on every page.
6. Materialize the logical value bytes in caller-owned or transaction-owned
   readback storage, depending on the established read API.

Any malformed live chain must fail the read with `Corruption`. Reads must not
return partial bytes.

### 3.3 Snapshot stability expectations

If a reader starts before a writer replaces or deletes an overflow-backed
value:

- the old reader continues to see the old overflow chain
- the writer publishes a new leaf snapshot that points somewhere else or
  removes the key
- the old chain remains reachable to that old reader until the read transaction
  closes
- the old chain may enter `pending_free`, but it must not become `reusable`
  while that older reader is still active

This is the core reason same-transaction reuse and eager pending promotion are
not allowed.

## 4. Replace, Delete, And Freeing Chains

Replacing or deleting overflow-backed values must update tree state and free
old chain pages atomically through the normal copy-on-write commit path.

### 4.1 Replace semantics

Replacement has three logical variants:

- overflow to overflow
- overflow to inline
- inline to overflow

For `overflow -> overflow`:

1. Allocate and encode the new overflow chain.
2. Clone the leaf path and rewrite the leaf slot to reference the new chain.
3. Enumerate the old chain exactly from the old leaf reference.
4. Add every old overflow page to `pending_free[current_txn_id]`.
5. Commit the new root, freelist image, and metadata atomically.

For `overflow -> inline`:

1. Encode the replacement value inline in the cloned leaf.
2. Enumerate the old overflow chain exactly.
3. Add the old chain pages to `pending_free[current_txn_id]`.
4. Commit through the normal tree plus freelist write path.

For `inline -> overflow`:

1. Allocate and encode the new overflow chain.
2. Rewrite the cloned leaf slot to the overflow reference form.
3. Free only the normal replaced tree pages through copy-on-write rules.
4. Do not invent any overflow free action for the old inline payload because it
   was stored inside the old leaf page body.

### 4.2 Delete semantics

Deleting an overflow-backed key must:

1. Locate the old leaf entry through the writer's base snapshot.
2. Decode and validate the old overflow reference.
3. Enumerate the exact old chain before removing the entry from the new tree
   image.
4. Add every old overflow page to `pending_free[current_txn_id]`.
5. Remove the key from the cloned leaf path.
6. Commit the new root and new freelist image together.

Delete must fail with `Corruption` if the old chain cannot be safely
enumerated. The engine must not remove the key and guess which pages to free.

### 4.3 Freeing chain rules

When a chain becomes unreachable in a committed write:

- all of its page IDs are grouped under the writer's future commit transaction
  ID
- the chain pages join the same `pending_free[current_txn_id]` accounting model
  used for tree and freelist pages
- the old chain must remain absent from `reusable` until a later transaction
  proves no older reader can still observe it

The freelist should not distinguish overflow pages from branch, leaf, or
freelist pages once they are safe to reuse.

## 5. Rollback Semantics

Rollback must preserve the M3 rule that uncommitted writes are discarded only
from transaction-local state.

### 5.1 Write rollback behavior

If a write transaction that allocated overflow pages rolls back before commit:

- the new leaf changes are discarded
- the new overflow chain pages are discarded from transaction-local ownership
- `pending_free` additions for replaced or deleted chains are discarded
- `DB.current_metadata` remains unchanged
- no old chain becomes logically freed because the replacement or delete never
  committed

### 5.2 Interaction with physical tail growth

The implementation may already have written private overflow pages to file
positions beyond the base snapshot during a failed or rolled-back write. That
does not make those pages committed.

Rules:

- rollback does not repair, truncate, or scan the file
- uncommitted tail pages remain ignored unless future committed metadata
  references them
- rollback does not publish those pages as reusable or pending
- a later recovery path must still trust only metadata-selected roots and
  freelist state

### 5.3 Failure during replace or delete

If replacement or deletion fails before commit because the old chain is corrupt
or a new chain cannot be built:

- the transaction may remain active only if the existing write error model
  allows continued safe use after that failure point
- otherwise the transaction should become terminal according to existing write
  failure rules
- in either case, the committed view must remain unchanged until a successful
  metadata publication happens

## 6. Tests

This feature needs focused coverage beyond the top-level M6 matrix.

### 6.1 Allocation and layout tests

Test cases:

- value just below the overflow threshold stays inline
- value just above the overflow threshold allocates one overflow page
- value spanning multiple overflow pages allocates the exact page count
- reusable pages are consumed before tail growth
- fragmented reusable pages can still satisfy a chain allocation
- same-transaction delete then large put does not reuse the just-freed pages

### 6.2 Snapshot read tests

Test cases:

- read transaction started before replace sees the old overflow value
- new read transaction started after commit sees the replacement value
- read transaction started before delete still sees the old overflow value
- reader uses its pinned `page_count` and rejects out-of-range overflow page IDs
- malformed overflow chain in a live snapshot returns `Corruption`

### 6.3 Replace and delete tests

Test cases:

- overflow to overflow replacement moves the old chain to pending
- overflow to inline replacement moves the old chain to pending
- inline to overflow replacement allocates the new chain and does not create a
  fake old overflow free
- delete of an overflow-backed key moves the full old chain to pending
- repeated large-value replacements eventually reuse eligible pages without
  unbounded logical growth

### 6.4 Rollback and failure tests

Test cases:

- put large value then rollback leaves no visible key
- replace overflow value then rollback leaves the old value visible
- delete overflow value then rollback leaves the key visible
- failure while building a new chain does not change the committed value
- delete or replace encountering a corrupt old chain fails without freeing
  guessed pages

### 6.5 Reuse and promotion tests

Test cases:

- old overflow pages stay pending while an older reader is active
- old overflow pages become reusable only after the blocking reader closes and a
  later writer promotes eligible pending groups
- a later large-value write reuses those pages before growing the tail
- reused pages can change page type safely after full overwrite and checksum
  regeneration

## 7. Acceptance

This feature can exit when all of the following are true:

- Overflow allocation follows the shared allocator order:
  eligible pending promotion, reusable reuse, then tail growth.
- The implementation never reuses pages freed by the active writer in that same
  transaction.
- Read transactions follow overflow references only through their pinned
  snapshot and never observe writer-private chain pages.
- Replacing an overflow-backed value with inline or overflow storage adds the
  exact old chain pages to `pending_free[current_txn_id]`.
- Deleting an overflow-backed value adds the exact old chain pages to
  `pending_free[current_txn_id]`.
- Old overflow pages do not become reusable until the M3 oldest-reader cutoff
  rule allows promotion.
- Rollback discards new chains, leaf rewrites, and pending-free deltas without
  changing committed metadata.
- Corrupt old chains cause replace/delete to fail instead of guessing pages to
  free.
- Tests cover allocation order, snapshot reads, replace/delete/freeing chains,
  rollback behavior, and later reuse after reader-safe promotion.
