# Copy-On-Write Mutation And Splits

This feature implements the M2 B+Tree mutation path for one unnamed ordered key
space. It covers `put` insert, `put` replace, `delete`, root-to-leaf path
cloning, leaf/root/branch splits, dirty-page staging, and the handoff seam that
lets M3 wrap the same logic in a real transaction API.

The implementation must preserve the M0 rule that visible root, branch, leaf,
freelist, and metadata pages are never overwritten in place. A successful M2
mutation becomes visible only when the inactive metadata slot is written, synced
as required by the selected durability mode, and then selected in memory.

## 1. Scope

In scope:

- `put` for missing keys.
- `put` replacement for existing keys.
- `delete` for existing and missing keys.
- Copy-on-write cloning of every page on the root-to-leaf mutation path.
- Leaf split, branch split, and root split.
- Delete without merge, redistribution, rebalance, or root collapse.
- Dirty-page staging for branch, leaf, and freelist pages.
- Tests for routing, persistence, split propagation, and failure boundaries.

Out of scope:

- Public transaction objects.
- Cursor and range-scan behavior.
- Overflow values.
- Freelist reuse as a performance feature.
- Crash fault injection beyond M2 write-failure state tests.
- Delete rebalance, merge, redistribution, compaction, or file shrinkage.

## 2. Core Data Flow

Mutation is split into two phases:

1. Build a staged tree mutation from a stable metadata snapshot.
2. Publish the staged result through the M2 simple write unit.

The tree mutation phase must not change selected metadata, page cache visibility,
or any committed page bytes. It returns a `MutationResult` containing:

- `new_root_page_id`
- ordered dirty branch and leaf pages
- a dirty freelist page if freelist ownership changes
- copied old page ids that became unreachable from the new root
- allocated tail page range and resulting `page_count`
- optional terminal state information for ambiguous write failure handling

The simple write unit owns page writes, syncs, metadata construction, inactive
metadata slot selection, in-memory metadata switching, and writer guard release.

## 3. Insert And Replace

`put(key, value)` follows the same mutation path for insert and replace:

1. Reject the call on a read-only handle.
2. Acquire the single writer guard.
3. Capture the selected metadata as the base snapshot.
4. Descend from `root_page_id` to the target leaf using normal branch routing.
5. Record a `TreePath` with each branch page, child index, separator context,
   and original child page id.
6. Clone the target leaf to a newly allocated page id.
7. Binary-search the cloned leaf entries.
8. If the key is missing, insert the encoded key/value entry.
9. If the key exists, replace the full encoded value.
10. If the cloned leaf fits, propagate the new child page id up the cloned path.
11. If the cloned leaf does not fit, split it and propagate a `SplitResult`.
12. Continue split propagation through cloned branch ancestors.
13. If the old root splits, allocate a new branch root.
14. Return a staged mutation result to the simple write unit.

Replacement is always treated as a write. A same-size, smaller, or larger value
must go through copy-on-write and metadata publication. A larger replacement that
overflows the leaf follows the same split path as a new insert.

The implementation must reject any key/value entry that cannot fit in an
otherwise empty leaf page. It must never truncate values, create overflow pages,
or encode partially written entries in M2.

## 4. Delete

`delete(key)` uses the same search and cloning discipline, but intentionally
does not rebalance:

1. Reject the call on a read-only handle.
2. Acquire the single writer guard.
3. Search for the key from the selected root.
4. If the key is missing, release the guard and return `false` without writing a
   new metadata generation.
5. Clone the root-to-leaf path to new page ids.
6. Remove the entry from the cloned leaf.
7. Re-encode cloned ancestors so each points at the new child page id.
8. Update separator keys only when required to keep branch routing correct.
9. Do not merge, redistribute, rebalance, or collapse the root.
10. Publish the new root through the simple write unit and return `true`.

Delete may leave underfull or empty non-root leaves. This is valid only when all
remaining keys are still reachable through normal point lookup and the checker
accepts the resulting branch ranges.

If the root is a leaf and the last entry is deleted, the new root remains an
empty leaf. If the root is a branch and all logical entries are deleted, M2 may
keep a branch-root shape with empty leaves, provided routing ranges remain
internally consistent and the checker deliberately accepts that shape.

## 5. Path Cloning

Path cloning must allocate new page ids for every page whose child pointer,
separator set, or leaf contents may change:

- clone the target leaf for every successful insert, replace, or delete
- clone every branch ancestor on the search path
- allocate split siblings from the tail
- allocate a new root when the old root splits
- allocate a new freelist page when pending-free ownership changes or the old
  freelist page itself must be copy-on-write

The old path pages remain visible to any reader pinned to the old metadata until
that metadata is no longer selected. They must be recorded as pending-free for
the new transaction id if the freelist codec supports pending-free groups.

The mutation builder should treat cloned pages as private objects until encoded
into `DirtyPage` records. Reads after failed mutation planning must continue to
use the old selected metadata and must not observe staged dirty pages.

## 6. Split Rules

All splits are byte-size based, not entry-count based.

General rules:

- keep both resulting pages non-empty
- target roughly half-full pages by encoded used bytes
- never split inside a key or value payload
- preserve strict key ordering
- copy promoted separator bytes into stable owned memory
- promote the first key of the new right page
- ensure every split result contains left child id, separator key, and right
  child id

Leaf split:

- distribute full key/value entries between left and right leaves
- preserve the reserved right-sibling field according to the M0 leaf format
- promote the first key of the right leaf to the parent
- make both leaf pages dirty with finalized page ids and later checksums

Root split:

- when a root leaf or root branch splits, allocate a new branch root
- encode one separator key in the new root
- point the left child to the split left page
- point the rightmost child to the split right page
- publish the new root page id only through metadata

Branch split:

- split separator keys and child pointers consistently with the branch routing
  rule: find the first separator greater than the search key; otherwise follow
  the rightmost child
- preserve the rightmost child pointer on both resulting branch pages
- promote the first key of the new right branch to the parent
- document in code whether the promoted separator remains encoded in the right
  branch; the codec, search, and checker must agree on that representation
- reject any branch state that cannot encode at least one separator and valid
  child pointers after split

## 7. Delete Routing Without Rebalance

M2 branch separators are routing lower bounds, not guaranteed copies of the
current first key in the right subtree after later deletes.

After delete, a stale separator is acceptable only if normal branch search still
sends every remaining key to the subtree that contains it. The mutation code must
refresh ancestor separators when deleting a boundary key would otherwise make a
remaining key unreachable.

The checker must validate descendant key ranges rather than requiring each
separator to equal the current minimum key of its right child. Underfull pages
and empty non-root leaves are not corruption by themselves.

## 8. Dirty Pages And Publication

Dirty pages move through the M0 lifecycle:

`cloned/new -> dirty -> written -> synced -> committed`

For each successful mutating operation:

1. Finalize dirty branch and leaf page bytes with page ids and checksums.
2. Finalize the new freelist page when needed.
3. Write all non-metadata pages to page ids outside the current visible state.
4. Run the required pre-metadata sync for the durability mode.
5. Build inactive metadata with incremented `generation` and `txn_id`, updated
   `root_page_id`, updated `freelist_page_id`, and updated `page_count`.
6. Write the inactive metadata slot.
7. Run the required metadata sync.
8. Switch selected in-memory metadata only after metadata write and required sync
   succeed.

If any page write or sync fails before the metadata switch, selected in-memory
metadata remains unchanged. If metadata write or metadata sync may have partially
completed, the handle enters the documented reopen-required state before later
writes can proceed.

## 9. Tests

Required unit and integration tests:

- insert into an empty root leaf
- insert many keys in sequential, reverse, and deterministic random order
- insert binary keys with embedded `0x00`, high-bit bytes, and prefix keys
- replace with same-size, smaller, larger, and leaf-splitting values
- reject an entry that is too large for an empty leaf
- delete a missing key without increasing metadata generation or txn id
- delete existing keys from root leaf, split leaves, and multi-level trees
- delete separator-boundary keys while preserving lookup routing
- delete the first key in a right subtree and verify remaining keys are reachable
- delete all keys from one non-root leaf and accept the underfull shape
- force leaf split and verify both sides after reopen
- force root split and verify branch routing after reopen
- force branch split with a small page-size fixture if practical
- compare deterministic random operations against an in-memory sorted map
- verify persistence after insert, replace, delete, root split, and branch split
- verify the checker catches unsorted leaves, duplicate keys, invalid child
  ranges, duplicate child pointers, wrong page types, invalid root state, and
  checksum failure
- verify non-metadata write failure leaves old selected metadata in memory
- verify ambiguous metadata write or sync failure enters the documented
  reopen-required state

Every persistence test should close and reopen the database, then assert point
lookup results and run the checker.

## 10. M3 Handoff

M2 must leave mutation planning separate from publication so M3 can wrap this
feature in explicit transactions without changing page semantics.

Required seams:

- the mutation builder accepts an immutable metadata snapshot
- the mutation builder returns a staged `MutationResult`
- dirty pages are owned by one write unit and are not globally visible
- allocation is abstracted behind a page allocator that can later support
  transaction-local state and freelist reuse
- metadata switching is isolated in the commit/write-unit layer
- commit failure terminal state is represented independently from tree logic
- tests can be reused for both auto-commit M2 writes and M3 explicit commits

M3 should be able to replace the single-operation auto-commit wrapper with a
write transaction object while preserving comparator behavior, page codecs,
split rules, delete routing, and checker range validation.

## 11. Acceptance

This feature is accepted when:

- `put` inserts missing keys using copy-on-write pages.
- `put` replaces existing values using the same mutation path.
- `delete` removes existing keys and returns `false` for missing keys without a
  new metadata generation.
- Leaf, root, and branch splits preserve lookup correctness after reopen.
- Delete never requires merge or rebalance, but remaining keys are still
  reachable through normal branch routing.
- Dirty branch, leaf, and freelist pages are written before metadata.
- Successful writes publish only through inactive metadata slot switching.
- Failed writes do not switch selected in-memory metadata or expose staged pages.
- Oversized inline entries fail deterministically.
- Checker validation accepts intentional underfull delete shapes and rejects real
  routing, ordering, checksum, and root-state corruption.
- Tests cover insert, replace, delete, path cloning, split propagation,
  no-rebalance routing, dirty-page failure boundaries, persistence, randomized
  correctness, and M3 handoff seams.
