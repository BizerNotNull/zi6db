# M5 Implementation Plan: Buckets And Cursor API

This plan implements the `M5: Buckets And Cursor API` milestone after M1-M4 have delivered a single-file database, ordered B+Tree storage, transactions, and crash-safe recovery. M5 must not weaken the M0 storage, transaction, metadata-switching, checksum, or copy-on-write rules.

## 1. Goal

M5 turns the crash-safe ordered key-value engine into a developer-usable embedded database API with:

- Multiple isolated logical collections.
- Bucket creation, deletion, lookup, and listing.
- Cursor movement with `first`, `last`, `seek`, `next`, and `prev`.
- Forward and backward range scans.
- Prefix scans.
- Predictable empty-bucket and missing-key behavior.

The public milestone language allows "buckets or tables". This plan uses `Bucket` as the implementation and documentation term. A later API may add `Table` aliases, but M5 should not maintain two separate concepts.

## 2. M0 Constraints M5 Must Preserve

- The database remains one file with fixed-size pages and positional I/O.
- The selected metadata page remains the only committed-state switch point.
- `metadata.root_page_id` continues to point at the root of the visible database state.
- M5 must not change the M0 page header, branch page, leaf page, metadata page, freelist page, checksum, or commit-order format.
- Visible pages are never modified in place; bucket catalog pages and bucket data pages use copy-on-write.
- Read transactions pin the metadata snapshot selected at `beginRead()`.
- Write transactions stage all bucket catalog, bucket data, root, and freelist changes privately until commit.
- A cursor lifetime is bound to its transaction.
- Closing, committing, or rolling back a transaction invalidates all buckets and cursors derived from it.
- A cursor from one transaction must never observe another transaction's uncommitted data.
- Commit success and failure semantics remain those defined by M0 and M4.

## 3. Data Model

### 3.1 Database Root As Bucket Catalog

M5 should interpret `metadata.root_page_id` as the root of a system bucket catalog B+Tree.

Catalog keys:

- Raw bucket names ordered lexicographically as bytes.
- Bucket names are unique.
- Empty bucket names are invalid.
- Names starting with the reserved byte prefix `0x00` are invalid for user buckets and reserved for future internal records.

Catalog values:

- A fixed-width little-endian bucket descriptor value, not an ad hoc serialized struct.
- `descriptor_version: u16`, initially `1`.
- `descriptor_kind: u16`, initially user bucket.
- `flags: u32`, initially `0`.
- `bucket_root_page_id: u64`.
- `entry_count: u64`, or `UINT64_MAX` when the implementation does not maintain an exact count in M5.
- Reserved `u64` fields that must be written as zero and rejected as `Corruption` if non-zero in a supported catalog version.

Each user bucket owns a separate B+Tree rooted at `bucket_root_page_id`. Bucket keys and values remain raw bytes ordered lexicographically.

### 3.2 Catalog On-Disk Compatibility

The catalog is a logical schema layered on top of the M0 B+Tree page format. It must not require a new normal page type or a different leaf/branch slot layout.

M5 must prevent accidental reinterpretation of a pre-M5 single-key-space tree as a bucket catalog:

- New M5-created databases must contain a catalog schema marker record using the reserved internal key prefix `0x00`.
- User bucket names must never be allowed to collide with internal catalog keys.
- `listBuckets()` must hide all internal catalog records.
- Opening an unmarked empty root tree may be treated as an empty catalog for developer builds, but the first write transaction that mutates buckets must insert the schema marker before publishing metadata.
- Opening an unmarked non-empty root tree must not silently treat existing key/value pairs as bucket descriptors. Until an explicit migration tool exists, return a clear compatibility error such as `IncompatibleVersion` or a dedicated schema-migration-required error.
- If the implementation has a reserved required feature flag or metadata state flag available for "bucket catalog root", M5 should set it for newly created catalog databases and validate it on open. The marker record is still required as an in-tree corruption check.
- Older engines are not guaranteed to understand M5 catalog files. The compatibility guarantee required for M5 is that the M5 engine never corrupts or misclassifies valid M0/M1-M4 files.

Catalog marker validation:

- The marker key must use the exact internal catalog marker chosen in code and documented next to the encoder.
- The marker value must include a catalog schema version, descriptor encoding version, and reserved-zero fields.
- A missing marker in a non-empty catalog root is a compatibility error, not an empty database.
- A malformed marker in a supported-format file is `Corruption`.
- A marker that names a newer required catalog schema is `IncompatibleVersion`.

### 3.3 Empty Bucket Representation

Creating a bucket allocates a new empty leaf page and stores its page id in the catalog.

An empty bucket:

- Has a valid leaf root page.
- Contains zero user entries.
- Returns no cursor item from `first()` or `last()`.
- Returns no item from `seek(any_key)`.

This avoids using `0` or another sentinel root page id that would violate M0 metadata and page validation expectations.

### 3.4 Bucket Isolation

Bucket operations must never mix key spaces:

- User keys are stored only inside the selected bucket B+Tree.
- Bucket names are stored only in the catalog B+Tree.
- Dropping one bucket must not modify any other bucket's root or user entries.
- A cursor created for one bucket must not be reusable against another bucket.

### 3.5 Compatibility With M2 Single-Key-Space Code

M2's B+Tree implementation should become the reusable tree primitive. M5 should add a catalog layer above it rather than duplicating B+Tree logic.

If earlier public examples expose a single raw key space, M5 may keep that API as a thin wrapper around an explicit default bucket only if it does not complicate the core implementation. The primary M5 acceptance path is the bucket API.

## 4. API Surface

### 4.1 Transactions And Buckets

Recommended shape:

```zig
pub fn getBucket(tx: *Tx, name: []const u8) !?Bucket;
pub fn createBucket(tx: *WriteTx, name: []const u8) !Bucket;
pub fn createBucketIfMissing(tx: *WriteTx, name: []const u8) !Bucket;
pub fn dropBucket(tx: *WriteTx, name: []const u8) !bool;
pub fn listBuckets(tx: *Tx, allocator: std.mem.Allocator) ![][]const u8;
```

Semantics:

- `getBucket()` returns `null` when the bucket is missing.
- `createBucket()` returns `BucketExists` or `InvalidState`-class error when the bucket already exists.
- `createBucketIfMissing()` returns the existing bucket when present, otherwise creates it.
- `dropBucket()` returns `true` when a bucket was dropped and `false` when it was missing.
- `listBuckets()` returns bucket names in lexicographic byte order.
- Mutating bucket-management calls require a write transaction.
- Bucket handles are invalid after their transaction closes.

The exact error names can be finalized during implementation, but tests must distinguish duplicate create, invalid name, read-only mutation, missing bucket, and closed transaction cases.

### 4.2 Bucket KV Operations

Recommended shape:

```zig
pub fn get(bucket: *Bucket, key: []const u8) !?[]const u8;
pub fn put(bucket: *Bucket, key: []const u8, value: []const u8) !void;
pub fn delete(bucket: *Bucket, key: []const u8) !bool;
pub fn exists(bucket: *Bucket, key: []const u8) !bool;
pub fn cursor(bucket: *Bucket) !Cursor;
```

Semantics:

- Empty keys are valid unless M2 already forbids them; M5 should not introduce a new restriction without a clear implementation reason.
- Empty values are valid.
- `get()` returns `null` for missing keys.
- `delete()` returns `false` for missing keys and `true` when it removes an existing key.
- `put()` replaces the existing value for the same key.
- Mutating key operations require a bucket from a write transaction.
- Within a write transaction, bucket reads and cursors created after a mutation must observe that transaction's own staged writes.
- Within a read transaction, bucket reads and cursors must observe only the metadata snapshot pinned when the transaction began.

### 4.3 Drop Semantics

`dropBucket()` is a catalog mutation plus a reachability change for every page owned by the dropped bucket.

Required behavior:

- Dropping a missing bucket returns `false` and must not dirty the catalog, allocate pages, or advance transaction state unnecessarily.
- Dropping an existing bucket in a write transaction removes the catalog entry from that transaction's staged catalog root.
- `getBucket()` in the same write transaction returns `null` for the dropped name after the drop, unless the bucket is recreated later in that same transaction.
- Bucket handles and cursors obtained before the drop become invalid immediately.
- Read transactions that began before the writer commits continue to see the old bucket and its old contents until those read transactions close.
- Rollback of the write transaction restores the pre-drop catalog view because no metadata switch occurred.
- Commit makes the drop visible atomically with the metadata switch; crash recovery selects either the old catalog or the new catalog according to M4 metadata rules.

Page ownership and freelist behavior:

- The drop implementation must traverse the dropped bucket's B+Tree and record every reachable branch and leaf page for `pending_free[new_txn_id]`.
- If M6 overflow values already exist when this code is revisited, overflow chains reachable from the dropped bucket must be included in the same pending-free group.
- Dropped pages must not be added directly to `reusable`.
- Dropped pages must not be reused by the same write transaction.
- Dropped pages must not be reused while any older read transaction can still reach the old bucket root.
- Recreating the same bucket name in the same write transaction allocates a new empty root page; it must not reuse or expose the dropped bucket's old root or entries.
- Dropping a bucket must not require rewriting pages belonging to other buckets.

## 5. Cursor Semantics

### 5.1 Cursor State

A cursor tracks:

- The transaction snapshot.
- The bucket root page id visible to that transaction.
- A bucket-local mutation revision for write transactions.
- A stack/path from root to current leaf.
- The current slot index.
- An invalid/no-item state.

`key()` and `value()` return `null` when the cursor is positioned off the end. Calls on a cursor invalidated by transaction close, bucket drop, or write-transaction mutation return `TransactionClosed` or `InvalidState` rather than silently returning `null`.

Returned key/value slices are valid only while the owning transaction remains open, the cursor remains on the same item, and no same-transaction mutation invalidates the cursor.

M5 chooses the conservative mutation rule:

- Read-transaction cursors are stable until the read transaction closes.
- Write-transaction cursors are invalidated by any `put()` or `delete()` in the same bucket because copy-on-write may relocate leaf pages.
- Write-transaction cursors are invalidated by `dropBucket()` for their bucket.
- Catalog cursors used by `listBuckets()` are invalidated by `createBucket()`, `createBucketIfMissing()` when it creates, or `dropBucket()`.
- A newly created cursor after the mutation must observe the write transaction's latest staged bucket root.
- Mutation-while-iterating can be optimized later, but M5 must not expose best-effort cursor recovery as a supported behavior.

### 5.2 Movement Rules

`first()`:

- Positions at the smallest key in the bucket.
- In an empty bucket, moves to no-item state.

`last()`:

- Positions at the largest key in the bucket.
- In an empty bucket, moves to no-item state.

`seek(target)`:

- Positions at the first key `>= target`.
- If every key is smaller than `target`, moves to no-item state.
- In an empty bucket, moves to no-item state.
- Does not require an exact match.

`next()`:

- If positioned on an item, moves to the next larger key.
- If currently at the last key, moves to no-item state.
- If already in no-item state, remains in no-item state.

`prev()`:

- If positioned on an item, moves to the next smaller key.
- If currently at the first key, moves to no-item state.
- If already in no-item state, remains in no-item state.

### 5.3 Boundary Behavior After Failed Searches

M5 should avoid hidden direction-dependent cursor recovery after a failed move:

- After `seek(target)` returns no item because `target` is beyond the last key, `prev()` remains no-item unless the API later explicitly supports "seek past end then prev".
- After `first()` on an empty bucket, both `next()` and `prev()` remain no-item.
- After `last()` on an empty bucket, both `next()` and `prev()` remain no-item.

If implementation wants `prev()` after failed `seek()` to land on the last key below the target, that should be a separate `seekLe()` or `seekLastLessOrEqual()` API, not implicit M5 cursor behavior.

## 6. Range And Prefix Scans

### 6.1 Range Scan API

Recommended shape:

```zig
pub const Bound = union(enum) {
    unbounded,
    included: []const u8,
    excluded: []const u8,
};

pub const Range = struct {
    start: Bound = .unbounded,
    end: Bound = .unbounded,
};

pub fn scanRange(bucket: *Bucket, range: Range) !RangeCursor;
```

Semantics:

- Forward range scans start at the first key satisfying `start`.
- Backward range scans start at the last key satisfying `end`.
- Every yielded key must satisfy both bounds.
- Empty ranges yield no items.
- `start > end` yields no items.
- `start == end` yields one item only when both bounds include the same existing key.
- Ranges must stream through cursor movement and must not materialize full result sets.

Bound handling must be byte-lexicographic and direction-independent:

- Forward start `.included(k)` uses `seek(k)`.
- Forward start `.excluded(k)` uses `seek(k)` and advances once if the positioned key equals `k`.
- Forward unbounded start uses `first()`.
- Backward end `.included(k)` positions at the greatest key `<= k`.
- Backward end `.excluded(k)` positions at the greatest key `< k`.
- Backward unbounded end uses `last()`.
- Backward positioning may be implemented with a helper such as `seek_le`/`seek_lt`; it must not depend on the public failed-`seek()` cursor state described above.
- Each `next()` or `prev()` result must be checked against both bounds before yielding, so leaf-boundary movement cannot leak an out-of-range key.
- Equal byte strings with any exclusive side produce an empty range.
- Empty byte-string bounds are valid keys and must be compared normally.

### 6.2 Prefix Scan API

Recommended shape:

```zig
pub fn scanPrefix(bucket: *Bucket, prefix: []const u8) !PrefixCursor;
```

Semantics:

- Prefix scans yield keys whose bytes start with `prefix`.
- Results are ordered lexicographically.
- Missing prefixes yield no items.
- An empty prefix scans the whole bucket.
- Prefix scans should be implemented as `seek(prefix)` plus a prefix check on each movement.
- If an exclusive upper bound is needed internally, compute the shortest lexicographic successor of the prefix when possible. If no successor exists, use an unbounded upper range and stop by checking `startsWith`.
- Prefix bytes are raw bytes, not text. Invalid UTF-8 and embedded `0x00` bytes in user keys are valid unless M2 forbids them globally.
- Prefix successor calculation must not overflow or truncate. For example, a prefix ending in one or more `0xff` bytes advances the nearest earlier byte that is not `0xff`; an all-`0xff` prefix has no finite successor.
- The implementation must always perform the final `startsWith(prefix)` check even when it also uses a computed upper bound.
- Backward prefix scans, if exposed in M5, must start at the greatest key with the prefix and then use `prev()` while checking `startsWith(prefix)`.

## 7. Execution Order

1. Add catalog marker and bucket descriptor encoding/decoding with strict length, version, and reserved-zero validation.
2. Add open-time compatibility handling for unmarked empty roots, unmarked non-empty roots, malformed markers, and newer catalog schema versions.
3. Reframe the metadata root as the catalog root in the in-memory database state.
4. Implement catalog B+Tree helpers for lookup, insert, delete, and ordered listing while filtering internal marker records.
5. Implement `getBucket()`, `createBucket()`, `createBucketIfMissing()`, `dropBucket()`, and `listBuckets()`.
6. Reuse the M2 B+Tree primitive for per-bucket `get`, `put`, `delete`, and `exists`.
7. Ensure write transactions copy-on-write both catalog pages and bucket data pages before mutation.
8. Add bucket and catalog mutation revisions so write-transaction cursors fail clearly after same-transaction mutation.
9. Implement drop traversal that records dropped bucket pages into `pending_free[new_txn_id]` without immediate reuse.
10. Implement cursor path traversal for `first`, `last`, `seek`, `next`, and `prev`.
11. Add internal lower-bound and upper-bound positioning helpers needed by forward and backward range cursors.
12. Add range cursor wrappers that enforce start and end bounds on every yielded item.
13. Add prefix cursor wrappers that enforce prefix matching without materializing results.
14. Add lifecycle checks for closed transactions, read-only mutation, dropped buckets, and invalid cursors.
15. Add persistence and snapshot tests across commit, rollback, crash-safe reopen, and concurrent readers.
16. Run existing M1-M4 tests to confirm the catalog layer did not weaken open, recovery, transactions, or commit ordering.

## 8. Test Matrix

| Area | Scenario | Expected result |
| --- | --- | --- |
| Catalog marker new DB | create M5 database | catalog marker exists, `listBuckets()` hides it |
| Catalog marker empty legacy | open unmarked empty root | treated as empty catalog or upgraded on first bucket write |
| Catalog marker non-empty legacy | open unmarked non-empty root | compatibility error; existing KV pairs are not reinterpreted |
| Catalog marker malformed | corrupt marker value in supported file | `Corruption` |
| Catalog marker newer schema | marker requires newer catalog schema | `IncompatibleVersion` |
| Descriptor decode | valid descriptor with reserved zeros | bucket opens with expected root page |
| Descriptor reserved bits | descriptor reserved field non-zero | `Corruption` |
| Descriptor root range | descriptor root page id out of range | `Corruption` or invalid catalog during open/check |
| Internal key filtering | catalog contains marker plus buckets | `listBuckets()` returns only user bucket names |
| Bucket create | create `a`, commit, reopen | `getBucket("a")` succeeds and bucket is empty |
| Bucket create duplicate | create `a` twice with `createBucket()` | second call returns duplicate-bucket error |
| Create if missing | call twice for `a` | both return usable bucket, only one catalog entry exists |
| Invalid bucket name empty | create empty bucket name | invalid-name error |
| Invalid bucket name reserved | create name beginning with `0x00` | invalid-name error |
| Bucket missing | `getBucket("missing")` | returns `null` |
| Bucket drop | create `a`, drop `a`, commit, reopen | `getBucket("a")` returns `null` |
| Drop missing | drop absent bucket | returns `false` and does not change catalog |
| Drop rollback | create `a`, commit, drop `a`, rollback | `getBucket("a")` still succeeds |
| Drop then recreate | drop `a`, recreate `a` in same write tx | new bucket is empty and old entries are absent |
| Drop handle invalidation | use old bucket handle after `dropBucket("a")` | returns invalid-state error |
| Drop cursor invalidation | use cursor from `a` after `dropBucket("a")` | returns invalid-state error |
| Drop reader snapshot | reader sees `a`, writer drops and commits | old reader still sees `a`; new reader does not |
| Drop pending-free | drop multi-page bucket | all dropped pages recorded in `pending_free[new_txn_id]`, not `reusable` |
| List buckets | create `b`, `a`, `c` | list returns `a`, `b`, `c` |
| Isolation | same key in `a` and `b` with different values | reads return bucket-specific values |
| Empty bucket first | cursor `first()` | key and value are `null` |
| Empty bucket last | cursor `last()` | key and value are `null` |
| Empty bucket seek | cursor `seek("x")` | key and value are `null` |
| Forward iteration | keys `a`, `b`, `c` | `first`, repeated `next` yields `a`, `b`, `c` |
| Backward iteration | keys `a`, `b`, `c` | `last`, repeated `prev` yields `c`, `b`, `a` |
| Seek exact | seek `b` in `a`, `b`, `c` | positions at `b` |
| Seek gap | seek `bb` in `a`, `b`, `c` | positions at `c` |
| Seek before first | seek `0` in `a`, `b` | positions at `a` |
| Seek after last | seek `z` in `a`, `b` | no item |
| Next at end | position at last then `next()` | no item |
| Prev at start | position at first then `prev()` | no item |
| Failed seek then prev | seek after last, then `prev()` | remains no item |
| Failed first then next | `first()` on empty, then `next()` | remains no item |
| Write cursor invalidation put | cursor in write tx, then `put()` same bucket | cursor operation returns invalid-state error |
| Write cursor invalidation delete | cursor in write tx, then `delete()` same bucket | cursor operation returns invalid-state error |
| Writer read-your-writes | put key, then create new cursor in same write tx | new cursor sees staged key |
| Reader no rebase | reader cursor open, writer commits update | reader cursor keeps old snapshot |
| Range inclusive | `[b, d]` over `a`..`e` | yields `b`, `c`, `d` |
| Range exclusive | `(b, d)` over `a`..`e` | yields `c` |
| Range unbounded start | `[..c]` | yields keys up to `c` by bound mode |
| Range unbounded end | `[c..]` | yields keys from `c` by bound mode |
| Range empty order | start greater than end | yields no items |
| Range equal included | `[b, b]` with `b` present | yields `b` |
| Range equal excluded | `(b, b)` | yields no items |
| Range lower excluded | `(b, d]` over `a`..`e` | yields `c`, `d` |
| Range upper excluded backward | backward `[b, d)` over `a`..`e` | yields `c`, `b` |
| Range empty byte bound | bound on empty key | compares as normal raw byte key |
| Range multi-leaf boundary | range crosses leaf split | yields exactly in-bound keys once |
| Prefix present | prefix `ab` over `aa`, `ab`, `aba`, `ac` | yields `ab`, `aba` |
| Prefix missing | prefix `ad` | yields no items |
| Prefix empty | prefix empty bytes | yields whole bucket |
| Prefix high bytes | prefix with no finite byte successor | still stops by `startsWith` check |
| Prefix embedded zero | prefix containing `0x00` in user keys | yields matching raw-byte keys |
| Prefix all `0xff` | prefix bytes all `0xff` | uses unbounded upper range and `startsWith` stop |
| Prefix false successor guard | computed successor would include non-prefix key | final `startsWith` check stops before non-prefix key |
| Snapshot reader | reader opens before writer creates bucket | reader does not see new bucket |
| Snapshot cursor | cursor opens before writer commits changes | cursor sees only pinned snapshot |
| Rollback | create bucket and put data, then rollback | bucket and data are absent |
| Read-only misuse | create/drop/put/delete in read tx | returns read-only transaction error |
| Transaction closed | use bucket or cursor after commit/rollback | returns transaction-closed or invalid-state error |
| Reopen persistence | create buckets and cursor through data after reopen | data and order are preserved |
| Crash matrix integration | inject failures during catalog and bucket page writes | recovery selects old or new metadata according to M4 rules |

## 9. Edge Cases To Specify During Implementation

- Maximum bucket name size should follow normal B+Tree key-size limits.
- Bucket descriptor values must fit inline until M6 large-value support exists.
- Catalog marker and bucket descriptor encoders must be deterministic byte-for-byte so crash and reopen tests can compare expected catalog contents.
- Catalog descriptor validation must reject short values, trailing bytes not owned by the schema version, non-zero reserved fields, unknown required flags, and root page ids outside `page_count`.
- If descriptor `entry_count` is not maintained in M5, it must be written as the documented unknown sentinel and no public count API should infer exact counts from it.
- Dropping a bucket must mark all pages reachable from that bucket root as pending-free according to M0/M6 freelist semantics. If M6 has not landed, the traversal can record pages for future reuse without immediate physical reclamation.
- Dropping a bucket while a read transaction still sees it must not reuse its pages until reader-safety rules allow reuse.
- A `Bucket` handle obtained before `dropBucket()` in the same write transaction should become invalid after the drop.
- Creating a bucket with the same name after dropping it in the same write transaction should create a new empty root and must not expose old entries.
- Cursor movement across leaf-page boundaries should work whether or not leaf right-sibling pointers are populated; parent-path traversal is acceptable for M5 correctness.
- Range and prefix cursors must treat byte slices as raw bytes; they must not use string collation, Unicode normalization, or sentinel terminators.
- Public cursor operations must distinguish no-item positioning from lifecycle invalidation so callers can tell normal iteration end from API misuse.

## 10. Non-Goals

- Nested buckets.
- Duplicate-key or multimap support.
- Custom comparators; ordering remains raw lexicographic byte order.
- SQL tables, schemas, indexes, or query planning.
- Full large-value overflow implementation beyond preserving M0 reserved hooks.
- Full free-space reuse policy beyond what M6 owns.
- Cross-process shared/exclusive locking beyond the M0/M7 boundary.
- Compaction, vacuum, backup, or statistics APIs.
- `mmap` or page-cache optimization work.
- Mutation-while-iterating guarantees beyond explicitly documented transaction-local behavior.

## 11. Acceptance Checklist

- M5 catalog files use only M0-compatible page, metadata, checksum, and commit-order structures.
- New catalog databases include a validated schema marker and deterministic descriptor encoding.
- Unmarked non-empty pre-M5 roots are rejected or routed to an explicit migration path, never silently reinterpreted as catalogs.
- Malformed catalog markers and descriptors fail with the correct `Corruption` or `IncompatibleVersion` class.
- Multiple buckets can be created, listed, reopened, and dropped.
- Bucket names are ordered lexicographically and duplicate names are rejected.
- Internal catalog records are hidden from user bucket listing and cannot be created as user buckets.
- Missing buckets and missing keys have clear nullable or boolean behavior.
- Bucket key spaces are isolated from each other.
- Dropping a bucket invalidates same-transaction handles and cursors, preserves older reader snapshots, and records dropped pages as pending-free rather than immediately reusable.
- Drop, recreate, rollback, commit, and crash recovery cases follow M0/M4 metadata-switching and freelist rules.
- Empty buckets produce no cursor item for `first`, `last`, or `seek`.
- `first`, `last`, `seek`, `next`, and `prev` are correct across single-leaf and multi-leaf trees.
- Failed cursor searches do not enable hidden direction-dependent recovery such as `seek(after_last)` then `prev()`.
- Forward and backward iteration stream results without materializing the full bucket.
- Range scans correctly enforce inclusive, exclusive, and unbounded bounds.
- Range scans correctly handle equal bounds, empty byte keys, backward scans, and multi-leaf boundaries.
- Prefix scans correctly handle present, missing, empty, embedded-zero, and high-byte prefixes.
- Read transactions keep stable bucket and cursor snapshots across concurrent writer commits.
- Write transactions provide read-your-writes for newly created bucket handles and cursors.
- Write-transaction cursors are invalidated by same-bucket mutation rather than silently continuing across relocated pages.
- Write transaction rollback removes staged bucket catalog and bucket data changes.
- Commit and recovery behavior for catalog changes follows the M0/M4 metadata-switching rules.
- Cursor, bucket, and transaction lifetime errors are tested.
- Existing M1-M4 create/open/recovery/transaction tests continue to pass.
