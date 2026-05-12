# M2 Implementation Plan

This document is based on `M2: Minimal Ordered KV Engine` in
[MILESTONES.md](../../../MILESTONES.md), the product goals in
[ROADMAP.md](../../../ROADMAP.md), the frozen M0 design under
[docs/design/m0/](../m0/README.md), and the expected M1 handoff in
[docs/design/m1/m1-implementation-plan.md](../m1/m1-implementation-plan.md).

M2 turns the M1 single-file storage skeleton into the first usable ordered
key-value engine. It should be useful enough to store, fetch, replace, delete,
reopen, validate, and benchmark raw byte keys and values, while deliberately
staying smaller than the later transaction, crash-recovery, bucket, cursor,
freelist-reuse, and overflow milestones.

## 1. Goal

M2 ships one logical ordered key space backed by the M0 B+Tree page format.

After M2, the engine should be able to:

- store raw byte keys in lexicographic order
- store inline raw byte values
- answer `get`, `exists`, `put`, and `delete`
- replace values for existing keys
- delete keys without requiring merge or rebalance
- split leaves and branch pages as the tree grows
- create a new root when the old root splits
- persist changes so point lookups remain correct after clean close and reopen
- run internal structural validation against reachable B+Tree pages
- provide a small benchmark harness for inserts and point lookups

M2 must preserve the on-disk format and copy-on-write direction frozen in M0.
It must not take a shortcut that makes M3 transactions or M4 crash safety require
a semantic redesign.

M2 is also the first milestone where mutation semantics can become confusing:
there is still no public transaction API, but each successful `put` or
successful `delete` must be an all-or-nothing visible state change at the
metadata boundary. A call may fail with an I/O or sync error, and M2 must not
claim M4 power-loss safety, but it also must never expose half-mutated in-memory
tree state to later calls.

## 2. Scope Boundary

### 2.1 In Scope

- Single unnamed key space rooted at the metadata `root_page_id`.
- B+Tree search over `PAGE_TYPE_BRANCH` and `PAGE_TYPE_LEAF`.
- B+Tree mutation for insert, replace, delete, leaf split, branch split, and root
  split.
- Variable-length inline key and value encoding using the M0 slot-directory page
  shapes.
- Tail-only page allocation for new B+Tree pages if freelist reuse is not ready.
- Copy-on-write mutation of the root-to-leaf path.
- A new freelist page per persisted write if needed to preserve M0 metadata and
  freelist ownership rules.
- Metadata switching for successful writes, using the inactive A/B metadata slot.
- Explicit failure handling for the internal one-operation write unit, including
  no in-memory metadata switch before the metadata write/sync path reports
  success.
- Internal checker coverage for page type, page id, checksum, key ordering,
  child ranges, root sanity, and basic reachability.
- Tests for API semantics, B+Tree shape changes, persistence after reopen, and
  malformed tree detection.
- A small benchmark harness for point lookup and insert workloads.

### 2.2 Explicitly Out Of Scope

- Public read-only or read-write transaction objects.
- Concurrent reader snapshots beyond whatever M1 already exposes internally.
- Fault-injection or power-loss crash testing.
- Multi-operation atomicity, rollback, or snapshot isolation exposed to callers.
- Fully balanced delete, page merge, page redistribution, or root collapse after
  large deletes.
- Freelist space reuse as a performance feature.
- Overflow pages for large values.
- Buckets, tables, nested namespaces, cursors, range scans, and prefix scans.
- Backward iteration.
- Compaction, vacuum, backup, statistics, or production integrity tooling.
- `mmap`, direct I/O, or an alternate storage backend.

## 3. M0 And M1 Dependencies

### 3.1 Hard M0 Dependencies

M2 must obey these frozen M0 decisions:

- Page `0` is the immutable header.
- Page `1` and page `2` are metadata slots A and B.
- Page `3+` are normal pages.
- All integer fields are little-endian.
- The default page size is `4096`, with the frozen min/max range.
- Normal pages use the common page header with `page_id`, `page_type`,
  `page_generation`, flags, and checksum.
- Branch pages use sorted separator keys, child page ids, a rightmost child, and
  variable-length key payloads.
- Leaf pages use sorted key/value entries, a reserved right-sibling field, slot
  flags, and variable-length key/value payloads.
- Metadata is the only on-disk commit switch point.
- Visible root, freelist, branch, leaf, and overflow pages are never overwritten
  in place.
- `generation` is the primary metadata ordering field.
- `txn_id` increments with `generation` on every committed write.
- Commit writes the inactive metadata slot and never rewrites the active slot.
- Metadata slot switching alternates strictly between A and B; M2 must not write
  both slots for one logical mutation or create equal-generation slots.
- The selected root and freelist pages must validate before open succeeds.
- Checksum failure makes the page invalid; the engine does not repair pages.

### 3.2 Expected M1 Handoff

M2 should start from the following M1 capabilities:

- `DB.create`, `DB.open`, and `DB.close`.
- Open/create options for at least `read_only`, `page_size`, `durability_mode`,
  and `lock_mode`.
- Frozen format constants in code.
- Header, metadata, common page, empty root leaf, and empty freelist codecs.
- Whole-page positional read/write helpers.
- File sync wrappers for the durability modes implemented by M1.
- Metadata selection and selected root/freelist validation on open.
- Conservative single-process writer guard behavior.
- Initial database shape:
  - root leaf page at page `3`
  - freelist page at page `4`
  - `page_count = 5`
  - metadata A valid with `generation = 1`
  - metadata B invalid

If M1 names modules differently from the examples in this document, M2 should
preserve the responsibility boundaries rather than forcing a rename-only refactor.

### 3.3 Handoff To Later Milestones

M2 should leave clear seams for later work:

- M3 should be able to wrap M2's internal write unit as a real write transaction
  without changing B+Tree page semantics. To make that possible, M2 should keep
  mutation planning separate from publication: tree mutation should produce a
  staged result, and the simple write unit should be the only layer that writes
  pages, writes metadata, switches in-memory metadata, and handles terminal
  commit failure state.
- M4 should be able to add fault injection around the same page write, sync, and
  metadata switch boundaries.
- M5 should be able to build buckets and cursors on top of the B+Tree traversal
  code without changing key ordering semantics.
- M6 should be able to replace tail-only allocation with freelist reuse and add
  overflow values without changing inline entry semantics.
- M7 should be able to reuse the checker as the basis for verify tooling.

## 4. Module Boundaries

The exact source tree can follow the shape established by M1. The following
boundaries describe responsibilities that should remain separate.

| Area | Suggested module | Responsibility |
| --- | --- | --- |
| Public KV surface | `src/kv.zig` or `src/db.zig` | `get`, `exists`, `put`, `delete`, option validation, public error mapping |
| Byte comparison | `src/tree/comparator.zig` | Lexicographic unsigned-byte ordering shared by search, mutation, and checker |
| Tree orchestration | `src/tree/btree.zig` | Root lookup, high-level insert/delete/search, split propagation |
| Page codecs | `src/tree/page_codec.zig` | Decode/encode leaf and branch bodies on top of the common page header |
| In-page operations | `src/tree/node.zig` | Binary search, insert entry, delete entry, replace entry, split node |
| Mutation planning | `src/tree/mutation.zig` | Copy-on-write path cloning, dirty page list, split result propagation |
| Simple write unit | `src/commit/simple_commit.zig` | M2 internal auto-commit flow, metadata switching, root/freelist publication |
| Page allocation | `src/storage/page_allocator.zig` | Tail allocation now, future freelist-backed allocation seam |
| Tree checker | `src/tree/checker.zig` | Reachability, ordering, checksum, child-range, and root sanity validation |
| Benchmarks | `bench/m2_kv.zig` or `test/bench_m2.zig` | Repeatable insert and point-lookup workloads |
| Tests | `test/` or module-adjacent tests | API, persistence, split, delete, corruption, and checker cases |

### 4.1 Dependency Direction

The dependency direction should stay simple:

- Public KV API calls into tree orchestration and the simple write unit.
- Tree orchestration calls page codecs, in-page operations, and storage page I/O.
- Page codecs depend on format constants and checksum helpers.
- Storage page I/O must not depend on tree logic.
- The checker may depend on codecs and storage reads, but production mutation code
  must not depend on test-only helpers.

### 4.2 Internal State Types

M2 should introduce explicit internal types rather than passing raw pages through
all layers:

- `KeyRef`: borrowed byte slice used only during one API call.
- `ValueRef`: borrowed byte slice accepted by `put`.
- `OwnedValue`: allocator-owned returned value from `get`, unless the API chooses
  a different explicit ownership model.
- `PageId`: the existing unsigned 64-bit logical page id.
- `TreePath`: ordered list of branch frames from root to leaf.
- `DirtyPage`: page id plus encoded page bytes ready for checksum finalization.
- `SplitResult`: promoted separator key plus left and right child page ids.
- `MutationResult`: new root page id, dirty pages, old copied pages, and
  page-count changes.
- `CheckReport`: structured internal validation result used by tests and later
  verify tooling.

## 5. KV API Semantics

M2 exposes the smallest useful API over one unnamed key space.

An implementation may choose exact Zig names that fit the M1 public surface, but
the behavior should match this shape:

```zig
pub fn get(db: *DB, allocator: Allocator, key: []const u8) !?[]u8;
pub fn exists(db: *DB, key: []const u8) !bool;
pub fn put(db: *DB, key: []const u8, value: []const u8) !void;
pub fn delete(db: *DB, key: []const u8) !bool;
pub fn check(db: *DB, options: CheckOptions) !CheckReport;
```

### 5.1 Key Semantics

- Keys are raw byte slices.
- Ordering is lexicographic over unsigned byte values.
- If two keys have equal bytes up to the shorter length, the shorter key sorts
  first.
- Duplicate keys are not allowed.
- Empty keys are valid unless implementation evidence shows they conflict with a
  frozen on-disk invariant; if rejected, the rejection must be explicit and
  documented before tests are written.
- Key bytes are copied into B+Tree pages on `put`; caller buffers may be reused
  after the API call returns.
- A key must fit inline in one leaf page together with its value and slot
  overhead. M2 does not spill keys or values to overflow pages.

### 5.2 Value Semantics

- Values are raw byte slices.
- Empty values are valid.
- `get` returns `null` when the key is missing.
- `get` must return a safe value with explicit ownership. The preferred M2
  behavior is to copy the value into caller-provided allocator memory, avoiding a
  page-cache lifetime contract before M3/M5.
- `put` replaces the full value when the key already exists.
- `put` has no append, compare-and-swap, duplicate-key, or partial-update mode.
- Values larger than the maximum inline leaf entry are rejected cleanly. M2 must
  not silently truncate or invent an overflow format beyond the M0 reservation.

### 5.3 Delete Semantics

- `delete` removes the key if present.
- `delete` returns `true` when an entry was removed.
- `delete` returns `false` when the key was already absent.
- Deleting an absent key is not corruption and should not change the file.
- Delete may leave underfull or empty non-root leaves.
- Delete does not have to merge, rebalance, redistribute, or shrink the tree.
- Delete must preserve lookup correctness for all remaining keys after reopen.
- Delete must not leave a separator/key-range state that can make a remaining
  key unreachable by normal point lookup.
- Delete may leave an otherwise empty branch-root tree after all keys are
  removed, but that representation must be accepted deliberately by the checker
  rather than treated as an accidental corruption escape hatch.

### 5.4 Read-Only And Lifecycle Semantics

- `get`, `exists`, and `check` are allowed on a read-only database handle.
- `put` and `delete` on a read-only database return `ReadOnlyTransaction` or the
  closest M1 error that already represents this M0 API rule.
- `close()` must still follow the M0 rule for active operations or future
  transactions.
- Public M2 calls should not expose snapshot semantics. Snapshot behavior belongs
  to M3.

### 5.5 Error Semantics

M2 should preserve the M0 error categories and add only deliberately documented
errors if needed.

At minimum:

- malformed pages found during open or checker validation map to `Corruption`
- incompatible format versions map to `IncompatibleVersion`
- write attempts on read-only handles map to `ReadOnlyTransaction`
- concurrent writer guard failures map to `WriteConflict` or `LockConflict`
- I/O, short write, short read, and sync failures map to `IoError`
- entry-size limit failures must be deterministic and documented, either as a
  dedicated M2 error such as `EntryTooLarge` or as an existing public error until
  the API error set is revised

## 6. B+Tree Design

### 6.1 Comparator

All tree layers must use one comparator:

1. Compare bytes as unsigned `u8`.
2. Return the first differing byte comparison.
3. If all compared bytes are equal, shorter length sorts first.
4. Equal length and equal bytes means the same key.

Tests should include binary keys with embedded `0x00`, high-bit bytes, prefix
relationships, empty values, and adjacent keys such as `a`, `aa`, and `ab`.

### 6.2 Leaf Page Invariants

A valid leaf page has:

- `page_type = PAGE_TYPE_LEAF`
- matching encoded `page_id`
- valid page checksum
- `entry_count` within page capacity
- slot directory entirely inside the page
- key and value payload ranges entirely inside the page
- no overlapping payload ranges unless the codec explicitly stores shared data,
  which M2 should avoid
- keys strictly sorted by the shared comparator
- no duplicate keys
- reserved right-sibling field preserved but not required for traversal
- value flags using only M2-supported inline-value bits

### 6.3 Branch Page Invariants

A valid branch page has:

- `page_type = PAGE_TYPE_BRANCH`
- matching encoded `page_id`
- valid page checksum
- `key_count` within page capacity
- separator keys strictly sorted by the shared comparator
- one child page id per separator for the subtree left of that separator
- one rightmost child pointer
- child page ids greater than or equal to `FIRST_DATA_PAGE_ID`
- no child page id equal to the branch page's own id
- no duplicate child page ids within the same branch
- no overflow or freelist pages as children

M2 should use this routing rule for branch pages:

1. Find the first separator key that is greater than the search key.
2. If found, descend into that slot's child page.
3. Otherwise, descend into the rightmost child.

Under this rule, a separator key is a lower bound for the subtree to its right.
Root splits and internal splits should promote the first key of the new right
page as the separator.

M2 treats branch separator keys as routing lower bounds, not necessarily as exact
copies of the current minimum key in the right subtree after later deletes. A
separator promoted by split should be the first key of the new right page at the
time of the split. After delete, keeping an older separator is valid only if it
still routes every possible lookup to the subtree whose key range contains that
lookup.

The checker must validate branch routing by key ranges, not by assuming each
separator is always equal to the current first key of the right child. That
distinction is required because M2 may delete without rebalance.

### 6.4 Search Algorithm

Search should be iterative or bounded-recursive:

1. Start at the selected metadata root page.
2. Validate page id, page type, and checksum before interpreting the page body.
3. On a branch page, binary-search separators and descend using the branch
   routing rule.
4. On a leaf page, binary-search entries.
5. Return found/missing plus the leaf position.

Search must never scan unreachable pages or infer state from file length.

### 6.5 Insert And Replace Algorithm

M2 should implement copy-on-write insertion:

1. Acquire the internal writer guard.
2. Capture the current selected metadata.
3. Descend from root to leaf, recording a `TreePath`.
4. Decode and clone every page on the mutation path.
5. Insert or replace the entry in the cloned leaf.
6. If the cloned leaf fits, write it as the new child for its parent.
7. If the cloned leaf overflows, split it into two leaves and produce a
   `SplitResult`.
8. Propagate splits up branch pages.
9. If the root splits, allocate a new branch root.
10. Re-encode every cloned ancestor whose child page id changed, even when no
    split occurred.
11. Finalize dirty page checksums.
12. Publish the new root through the M2 simple write unit.

Replacement is a mutation even when the key already exists. If the replacement
value changes the encoded size enough to overflow the page, it follows the same
split path as insert.

### 6.6 Split Rules

Split logic should optimize for correctness and simplicity:

- Split by encoded byte size, not only by entry count.
- Keep both sides non-empty.
- Prefer an approximately half-full split by used bytes.
- Do not split inside a key or value payload.
- Reject an entry that cannot fit in an otherwise empty leaf page.
- Promote the first key of the right page.
- For branch splits, preserve the branch routing rule and rightmost child.
- Branch split must document whether the promoted separator remains in the new
  right branch or is removed from it; either choice is acceptable only if the
  branch encoding, routing rule, and checker child-range validation agree.
- Copy promoted separator bytes into the parent; do not borrow from temporary
  buffers that will be freed.

### 6.7 Delete Algorithm

M2 delete should be intentionally conservative:

1. Acquire the internal writer guard.
2. Search for the key.
3. If missing, return `false` without writing a new metadata generation.
4. Clone the root-to-leaf path.
5. Remove the entry from the cloned leaf.
6. Re-encode cloned ancestors so they point to the new child page ids.
7. Do not merge or rebalance underfull pages.
8. Update ancestor separator keys only when required to preserve routing ranges;
   stale separators are acceptable lower bounds only if every remaining key is
   still reachable through normal branch search.
9. Allow empty non-root leaves if their existing separator ranges still route
   lookups correctly.
10. If the root is a leaf and becomes empty, keep it as an empty leaf.
11. If the root is a branch and all logical entries are deleted, M2 may keep the
    branch root and empty leaves instead of collapsing it, but search and checker
    logic must treat that shape as a valid empty tree.
12. Publish the new root through the M2 simple write unit.

Because M2 allows underfull pages, the checker should distinguish corruption from
accepted underfull states. Too few entries is not a checker failure in M2 unless
it breaks routing, ordering, reachability, or root sanity.

### 6.8 Root Rules

- A fresh M1 database starts with a root leaf.
- The root may be a leaf or branch.
- The root page id must match selected metadata.
- A root leaf may be empty.
- A root branch must have at least one separator and a valid rightmost child.
- Root split creates a new branch root with one separator and two children.
- M2 does not need to collapse a branch root after deletes, as long as remaining
  lookups are correct.

## 7. Persistence And Write Flow

M2 does not expose full transactions, but every successful mutating API call
should behave like a one-operation internal write unit.

The word "commit" in this section refers only to that internal write unit. It is
not a public transaction API, does not provide multi-operation atomicity, and
does not provide M4 crash-fault guarantees beyond following the M0 ordering
shape for clean writes.

### 7.1 Internal Auto-Commit Write Unit

For `put` and successful `delete`, use this flow:

1. Reject the operation if the database handle is read-only.
2. Acquire the single writer guard.
3. Capture the selected metadata as the base state.
4. Build the B+Tree mutation using copy-on-write pages.
5. Allocate new page ids from the tail unless safe freelist reuse already exists.
6. Build a new freelist page that preserves existing freelist contents and records
   copied old pages as pending-free where the M0 format requires it.
7. Finalize checksums for dirty branch, leaf, and freelist pages.
8. Write all non-metadata pages.
9. Use the M1/M0 sync wrapper for the pre-metadata sync required by the selected
   durability mode.
10. Build metadata in the inactive slot with incremented `generation` and
    `txn_id`, updated `root_page_id`, updated `freelist_page_id`, and updated
    `page_count`.
11. Write the inactive metadata page.
12. Use the M1/M0 metadata sync wrapper required by the selected durability mode.
13. Switch the in-memory selected metadata only after the metadata write path
    reports success.
14. Release the writer guard.

M2 should follow this sequence even if M4 later adds stronger crash-fault tests.
If a platform path does not yet provide the full M0 `strict` durability behavior,
the M2 implementation and PR notes must state the limitation clearly and must not
claim M4 crash safety.

### 7.1.1 Failure Semantics Without Full Transactions

M2 must make the internal write unit failure states explicit:

- If the mutation fails before any page write, return the error, release the
  writer guard, and leave the in-memory selected metadata unchanged.
- If writing a non-metadata page, pre-metadata sync, metadata write, or metadata
  sync fails, return `IoError`, release the writer guard, and leave the in-memory
  selected metadata unchanged.
- If metadata write may have started but metadata sync did not report success,
  the on-disk outcome is ambiguous until reopen. The handle should enter a
  conservative "reopen required before further writes" state so a later M2 write
  cannot overwrite the same inactive metadata slot while recovery might select
  the newer on-disk metadata.
- Reads after an ambiguous failed write may either continue against the old
  in-memory metadata snapshot or fail with `InvalidState`, but the chosen policy
  must be documented and tested. They must not read from partially staged dirty
  pages.
- No failed write may publish a new root, freelist, `generation`, or `txn_id` in
  memory before the metadata write and required sync have succeeded.
- Reopen remains the authoritative way to determine whether old or new metadata
  is selected after an ambiguous metadata-write failure.

These rules intentionally mirror the M0 commit-failure classes while keeping M2
smaller than the M3 transaction API.

### 7.2 Allocation Policy

The recommended M2 allocation policy is tail-only:

- Allocate every cloned page, split page, new root, and new freelist page from the
  current `page_count` tail.
- Increase `page_count` in the new metadata only after all allocated pages are
  known.
- Do not reuse pages from `reusable` yet unless M6-level safety checks already
  exist.
- Do not infer reusable pages from unreachable file tail pages.
- Preserve existing freelist data when writing the new freelist page.

This policy grows files quickly, but it is simple and keeps M2 from weakening M0
copy-on-write or reader-safety rules.

### 7.3 Freelist Interaction

M2 is not a space-reuse milestone, but it must not corrupt the freelist contract.

At minimum:

- The selected freelist page remains reachable from metadata.
- A successful write publishes a new freelist page if freelist state changes or
  if the old freelist page itself must be copy-on-write.
- Old copied pages from the previous root path and old freelist page should enter
  `pending_free[new_txn_id]` if the M0 freelist codec is ready.
- Pages freed by logical delete are not reused by the same write.
- M2 tests should not require file size shrinkage.

If the M1 freelist codec is still only an empty-page placeholder, M2 should add
the smallest encoding needed to preserve pending-free groups without implementing
reuse.

## 8. Structural Checker

M2's checker is an internal correctness tool first and a future M7 verify seed
second.

### 8.1 Checker Inputs

The checker should accept:

- a database handle or storage reader
- a metadata snapshot to check, defaulting to the selected metadata
- options for fast root-only checks versus full tree checks

### 8.2 Checker Coverage

The full M2 checker should validate:

- selected metadata root page id is in range
- root page type is `LEAF` or `BRANCH`
- every reachable page checksum validates
- every reachable page id matches its physical page id
- every reachable page has an allowed page type
- leaf keys are strictly sorted
- branch separator keys are strictly sorted
- branch child pointers are in range
- branch child pointers are unique within the reachable tree
- child page types are valid for tree traversal
- branch routing ranges are respected by descendant leaf keys
- branch separators are valid lower bounds for right-side ranges, while allowing
  stale lower bounds left by accepted no-rebalance deletes
- all reachable leaves have the same depth, unless M2 deliberately documents a
  weaker accepted tree shape
- no reachable tree page is visited twice
- no branch points to itself
- key/value payload offsets and lengths stay within page bounds
- slot directories do not overlap invalid regions
- root sanity rules are satisfied
- an empty logical tree represented by a branch root and empty leaves is accepted
  only if every branch range remains internally consistent

### 8.3 Checker Non-Goals

The M2 checker does not need to:

- prove crash recovery outcomes
- verify the complete freelist reachability model
- require minimum page occupancy after deletes
- verify overflow chains
- validate bucket catalogs
- perform repair

## 9. Test Matrix

Every M2 implementation PR should include automated tests mapping to the matrix
below. Names should describe behavior, not implementation details.

| Area | Cases |
| --- | --- |
| Empty database | `get` missing key returns null, `exists` returns false, `delete` missing returns false |
| Basic put/get | one key, many keys, binary keys, empty key if supported, empty value |
| Ordering | sequential inserts, reverse inserts, random inserts, prefix keys, high-bit bytes |
| Replacement | same-size value, smaller value, larger value, replacement causing leaf split |
| Delete | existing key, missing key, first key in leaf, last key in leaf, all keys from root leaf |
| Delete routing | delete separator-boundary keys, delete the first key in a right subtree, delete all keys from one non-root leaf, delete all keys from a multi-level tree |
| Underfull delete | delete many keys after splits; remaining keys still resolve correctly and accepted empty leaves still pass checker range validation |
| Leaf split | enough inserts to split a root leaf; both sides are searchable after reopen |
| Root split | enough inserts to create a branch root; branch routing is correct after reopen |
| Branch split | enough inserts to create at least three levels if practical with small page fixtures |
| Variable length | short keys, long keys, short values, long inline values, mixed sizes |
| Size limits | maximum inline entry succeeds, entry too large fails cleanly |
| Persistence | reopen after put, replace, delete, root split, and mixed workloads |
| Metadata switch | generation and txn id increment after successful writes; missing-key delete does not publish a new generation |
| Write failure | injected or mocked non-metadata write failure leaves old in-memory metadata selected; ambiguous metadata write/sync failure requires documented reopen behavior |
| Freelist handoff | new freelist page remains reachable from metadata; copied old pages are preserved as pending-free if the codec supports it; file size is not expected to shrink |
| Read-only | read operations succeed; put/delete fail on read-only handle |
| Checker success | full checker passes after representative workloads |
| Checker failures | unsorted leaf, duplicate key, wrong child page type, bad child range, duplicate child pointer, wrong checksum, invalid root type |
| Fuzz/property | deterministic random operation sequence compared with an in-memory sorted map |
| Bench smoke | insert and point-lookup harness runs without asserting performance thresholds |

### 9.1 Persistence Assertions

Persistence tests should close and reopen the file, then assert:

- all expected keys are present
- all deleted keys are absent
- replacement values are visible
- old values are not visible
- metadata selects the newest valid generation
- the checker passes after reopen

### 9.2 Randomized Correctness

Add a deterministic randomized test with a fixed seed:

1. Maintain an in-memory sorted reference map.
2. Generate `put`, `delete`, `get`, and `exists` operations.
3. Apply every operation to both the reference map and `zi6db`.
4. Reopen periodically.
5. Compare every key in the reference map using point lookups.
6. Probe deleted and never-inserted keys as well as present keys.
7. Run the checker after each reopen or at the end of each operation batch.

This test is especially valuable because M2 intentionally defers delete
rebalancing; randomized workloads are likely to find stale routing bugs.

## 10. Benchmark Harness

M2 should add a small benchmark harness without turning performance tuning into
the milestone goal.

Required workloads:

- sequential insert of fixed-size keys and values
- random insert of fixed-size keys and values
- point lookup of existing keys
- point lookup of missing keys
- mixed replace workload

Required scope limits:

- use inline values only
- use one database file and one logical key space
- run with at least one small fixture size that forces splits quickly and one
  default-page-size fixture for realistic smoke coverage
- keep the harness deterministic by accepting a seed for random workloads
- exclude cursors, range scans, overflow values, freelist reuse tuning, and
  cross-process concurrency

Recommended reporting:

- operation count
- elapsed time
- operations per second
- database file size
- page count
- tree height if checker or tree stats can compute it cheaply
- random seed and key/value size parameters

The benchmark should be easy to run locally, but M2 acceptance should not depend
on absolute throughput numbers or comparisons with other databases. Treat large
regressions discovered by the harness as debugging signals, not as a reason to
expand M2 into a performance tuning milestone.

## 11. Recommended Execution Order

To reduce rework and isolate failures, implement M2 in this order:

1. Add the shared byte comparator and tests.
2. Complete leaf and branch page codecs on top of the M1 common page header.
3. Add in-page binary search and leaf insert/replace/delete helpers.
4. Implement read-only tree search over a root leaf and branch pages.
5. Expose `get` and `exists` for the initial M1 empty root leaf.
6. Add tail-only page allocation helpers for M2 dirty pages.
7. Implement copy-on-write insertion without split for a single leaf.
8. Add leaf split and root split.
9. Add branch insertion and branch split propagation.
10. Implement value replacement through the same mutation path.
11. Implement conservative delete without merge or rebalance.
12. Add the M2 simple write unit and metadata publication path.
13. Add write-failure state tests for non-metadata write failure and ambiguous
    metadata write/sync failure.
14. Add reopen persistence tests after every mutation class.
15. Add the checker and wire it into tests.
16. Add deterministic randomized tests against an in-memory sorted map.
17. Add the benchmark harness.
18. Document local build, test, and benchmark commands in the implementation PR.

This order keeps codecs and read-only search stable before mutation logic, then
adds persistence only after the tree mutation result can be represented clearly.

## 12. Acceptance Checklist

M2 is complete only when all of the following are true:

- `get`, `exists`, `put`, and `delete` are available for one logical key space.
- Keys are compared with one documented lexicographic byte comparator.
- Duplicate keys cannot exist in a leaf or across reachable leaves.
- `put` inserts missing keys.
- `put` replaces existing values.
- `delete` removes existing keys and reports missing keys without writing a new
  commit generation.
- Variable-length inline keys and values are encoded and decoded correctly.
- Oversized entries are rejected clearly instead of corrupting a page.
- Leaf split works.
- Root split works.
- Branch split works or is explicitly covered by a test fixture that forces it.
- Delete may leave underfull or empty non-root leaves, but lookup correctness,
  branch routing ranges, and checker validation remain intact.
- Mutations do not overwrite the current visible root, branch, leaf, or freelist
  pages in place.
- Successful writes publish a new root through metadata, not through header
  mutation.
- Failed writes do not switch in-memory metadata, do not expose dirty staged
  pages to later reads, and document the reopen-required policy for ambiguous
  metadata-write failures.
- `generation` and `txn_id` increase together on successful mutating writes.
- Clean close and reopen preserve inserted, replaced, and deleted key state.
- The checker catches malformed page types, unsorted keys, duplicate keys,
  duplicate child pointers, invalid child ranges, invalid root state, and
  checksum failures covered by M2.
- Tests cover single-leaf, split, multi-level, replacement, delete, persistence,
  failure-state, and randomized workloads.
- The benchmark harness can run deterministic point lookup, insert, and replace
  workloads without imposing absolute throughput targets.
- The implementation leaves obvious seams for M3 transactions, M4 fault
  injection, M5 cursors/buckets, M6 overflow/freelist reuse, and M7 verification,
  especially by separating mutation planning from metadata publication.

## 13. Non-Goals

Do not expand M2 to include:

- public transaction API design beyond the M0 sketch
- multiple logical buckets or tables
- cursor API or ordered iteration
- range scans or prefix scans
- merge or rebalance after delete
- reclaiming file space as a user-visible feature
- overflow page storage for large values
- crash-safe fault-injection claims
- multi-process locking hardening
- compaction, backup, or statistics APIs
- absolute performance targets

## 14. Exit Outcome

If M2 succeeds, `zi6db` becomes a real but intentionally small embedded ordered
KV engine. It will not yet be a transactionally complete or crash-hardened
database, but it will have the hardest storage-engine foundation in place: a
copy-on-write B+Tree, stable raw-byte KV semantics, reopen persistence,
structural validation, and a mutation path that later milestones can extend
without changing the format frozen in M0.
