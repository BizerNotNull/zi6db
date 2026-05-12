# Bucket Management API

This document defines the concrete M5 implementation plan for the `bucket
management API` feature. It refines
[m5-implementation-plan.md](/D:/代码D/zi6db/docs/design/m5/m5-implementation-plan.md)
and must preserve the transaction, copy-on-write, and metadata-switch rules
already established by M3 and M4.

The feature is the public control plane for bucket lifecycle. It does not
define cursor movement, range scans, or prefix scans beyond the bucket-listing
behavior needed to enumerate user buckets.

## 1. Scope

This feature covers:

- `getBucket(tx: *Tx, name: []const u8) !?Bucket`
- `createBucket(tx: *WriteTx, name: []const u8) !Bucket`
- `createBucketIfMissing(tx: *WriteTx, name: []const u8) !Bucket`
- `dropBucket(tx: *WriteTx, name: []const u8) !bool`
- `listBuckets(tx: *Tx, allocator: std.mem.Allocator) ![][]const u8`
- bucket name validation
- transaction and lifecycle rules for bucket handles
- catalog filtering for internal records
- persistence, rollback, and crash-safety test coverage
- M6 handoff expectations for stats and future storage work

This feature does not cover:

- bucket KV operations such as `put`, `delete`, or `cursor`
- range or prefix cursor APIs
- nested buckets
- large-value overflow behavior beyond preserving M6 hooks
- migration tooling for pre-M5 single-key-space databases

## 2. Inputs And Frozen Constraints

The implementation must preserve these decisions from the M5 and M3 plans:

- `metadata.root_page_id` points to the bucket catalog root
- the catalog is stored in the existing B+Tree page format, not a new page type
- committed state still switches only by writing the inactive metadata page
- write transactions stage catalog changes privately until commit
- read transactions pin one metadata snapshot and never rebase
- only a write transaction may create or drop buckets
- visible pages are never modified in place
- dropped bucket pages enter `pending_free[new_txn_id]`, not `reusable`
- bucket handles and any future bucket-local cursors are invalid after
  transaction close
- internal catalog records remain hidden from user bucket APIs

## 3. API Surface

### 3.1 Public Methods

The M5 bucket-management surface is:

```zig
pub fn getBucket(tx: *Tx, name: []const u8) !?Bucket;
pub fn createBucket(tx: *WriteTx, name: []const u8) !Bucket;
pub fn createBucketIfMissing(tx: *WriteTx, name: []const u8) !Bucket;
pub fn dropBucket(tx: *WriteTx, name: []const u8) !bool;
pub fn listBuckets(tx: *Tx, allocator: std.mem.Allocator) ![][]const u8;
```

Required results:

- `getBucket()` returns `null` when the bucket is missing
- `createBucket()` returns a duplicate-bucket error when the bucket already
  exists
- `createBucketIfMissing()` returns an existing bucket when present and creates
  one otherwise
- `dropBucket()` returns `true` only when an existing bucket was removed
- `listBuckets()` returns only user bucket names in lexicographic byte order

### 3.2 Bucket Handle Rules

A returned `Bucket` should be a transaction-bound handle with at least:

- a pointer to the owning transaction
- the validated bucket name or a reference to stable copied name bytes
- the visible `bucket_root_page_id`
- a bucket-local mutation revision or invalidation token for write transactions
- a dropped/invalid flag for same-transaction lifecycle checks

The handle must not cache mutable DB-global metadata. It should behave like a
view over one transaction snapshot plus one catalog entry.

## 4. Name Validation

Bucket names are raw bytes, not text. The validation policy should stay small,
deterministic, and compatible with the M5 catalog rules.

### 4.1 Required Validation Rules

The implementation must reject:

- empty names
- names whose first byte is `0x00`
- names that exceed the maximum key length accepted by the underlying B+Tree
- names that cannot be copied or encoded because of allocator failure

The implementation must allow:

- arbitrary non-empty byte strings that do not begin with `0x00`
- invalid UTF-8
- embedded `0x00` bytes after the first byte
- names that differ only by byte value, not normalized text rules

### 4.2 Validation Timing

Validation should happen before any catalog mutation or page allocation.

Rules:

- `getBucket()` validates the requested name before catalog lookup
- `createBucket()` and `createBucketIfMissing()` validate before checking for an
  existing entry
- `dropBucket()` validates before mutation logic
- `listBuckets()` never revalidates persisted names as user input; it filters
  internal records and treats malformed stored names as corruption in the
  catalog-validation layer

## 5. Transaction Rules

### 5.1 Read And Write Requirements

Bucket management operations follow these transaction rules:

- `getBucket()` is valid in read and write transactions
- `listBuckets()` is valid in read and write transactions
- `createBucket()`, `createBucketIfMissing()`, and `dropBucket()` require a
  live write transaction
- calling a mutating bucket-management API through a read transaction returns
  `ReadOnlyTransaction`
- calling any bucket-management API on a closed transaction returns
  `TransactionClosed`

### 5.2 Snapshot Rules

Snapshot visibility must match the M3 transaction model:

- a read transaction sees the catalog root pinned at `beginRead()`
- a write transaction starts from its base catalog root and reads its own staged
  catalog updates
- a reader that began before a writer commits does not see newly created or
  dropped buckets
- a new reader that begins after commit sees the committed catalog state
- rollback discards staged bucket creation and drop state completely

### 5.3 Same-Transaction Lifecycle Rules

The implementation should use explicit invalidation rather than best-effort
recovery.

Rules:

- a bucket handle becomes invalid when its transaction commits or rolls back
- a bucket handle returned before `dropBucket(name)` in the same write
  transaction becomes invalid immediately after the drop
- if the same name is recreated later in that write transaction, a new bucket
  handle must be returned; the old handle stays invalid
- `getBucket()` after `dropBucket()` in the same write transaction returns
  `null` until the bucket is recreated
- `createBucketIfMissing()` after dropping the same name in the same write
  transaction creates a new empty bucket rooted at a new page

## 6. Operation Semantics

### 6.1 `getBucket()`

Implementation rules:

1. validate the supplied name
2. read the visible catalog root for the transaction
3. look up the exact bucket name in the catalog tree
4. return `null` when the user bucket is absent
5. decode and validate the bucket descriptor before constructing the handle

`getBucket()` must not allocate a new root page, write the catalog marker, or
change transaction state.

### 6.2 `createBucket()`

Implementation rules:

1. validate the supplied name
2. confirm the transaction is an active writer
3. ensure the catalog marker exists or stage it before the first catalog write
4. check whether the name already exists in the staged catalog view
5. return a duplicate-bucket error if it does
6. allocate a new empty leaf page for the bucket root
7. encode the initial descriptor with that root page id
8. insert the catalog entry by copy-on-write tree mutation
9. return a handle bound to the new staged bucket root

The new bucket must be empty and isolated from every other bucket.

### 6.3 `createBucketIfMissing()`

Implementation rules:

1. validate the supplied name
2. confirm the transaction is an active writer
3. read the staged catalog view
4. if the bucket exists, return a handle to the existing staged bucket
5. otherwise perform the same steps as `createBucket()`

This API must be idempotent within one write transaction except when the caller
has already dropped the bucket in that same transaction. In that case, the call
creates a new empty bucket.

### 6.4 `dropBucket()`

Implementation rules:

1. validate the supplied name
2. confirm the transaction is an active writer
3. check the staged catalog view for the named bucket
4. return `false` immediately if the bucket is absent
5. remove the catalog entry by copy-on-write mutation
6. mark all same-transaction handles for that bucket invalid
7. traverse the dropped bucket tree and record every reachable page in
   `pending_free[new_txn_id]`
8. return `true`

Required safety rules:

- dropping a missing bucket must not dirty the catalog
- dropped pages must not become reusable in the same transaction
- dropped pages must not be exposed through a later same-name recreation
- dropping one bucket must not rewrite or merge data pages from other buckets

### 6.5 `listBuckets()`

Implementation rules:

1. read the transaction-visible catalog root
2. iterate catalog records in lexicographic byte order
3. skip internal marker or reserved internal records
4. copy user bucket names into caller-owned memory
5. return the copied name list in encounter order

Observable behavior:

- internal keys are never returned
- results are sorted by raw byte order
- an empty catalog returns an empty list
- write transactions list their own staged catalog mutations

## 7. Catalog Integration And Invalidation

This feature depends on the catalog layer already defined by the M5 plan.

Implementation guidance:

- factor catalog lookup, insert, delete, and ordered iteration into internal
  helpers shared by all five APIs
- keep bucket-name validation at the public API edge, not hidden in low-level
  tree traversal
- keep catalog mutation revision tracking separate from future per-bucket KV
  mutation revision tracking
- make `listBuckets()` depend on ordered catalog iteration but not on the later
  public cursor API

For write transactions, maintain a catalog mutation revision counter:

- increment on `createBucket()`
- increment on `createBucketIfMissing()` only when it creates a bucket
- increment on `dropBucket()` only when it removes a bucket

The revision lets same-transaction bucket and catalog-derived views fail
predictably after a lifecycle-changing mutation.

## 8. Implementation Slices

Implement this feature in the following order:

1. add a shared bucket-name validator and error mapping
2. add bucket-handle lifecycle checks and invalidation tokens
3. add internal catalog helpers for lookup, insertion, deletion, and ordered
   listing with internal-record filtering
4. implement `getBucket()` on top of catalog lookup and descriptor validation
5. implement `createBucket()` including empty-root allocation and marker staging
6. implement `createBucketIfMissing()` as a staged lookup plus conditional
   create path
7. implement `dropBucket()` including bucket-tree reachability traversal into
   `pending_free[new_txn_id]`
8. implement `listBuckets()` with caller-owned name copies
9. add commit, rollback, reopen, and concurrent-reader tests
10. run the existing M3 and M4 transaction and recovery tests to confirm the
   bucket-management layer did not weaken snapshot or commit semantics

## 9. Test Plan

Minimum tests for this feature:

| Area | Scenario | Expected result |
| --- | --- | --- |
| get missing | `getBucket("missing")` | returns `null` |
| create | create `a`, commit, reopen | `getBucket("a")` succeeds and bucket is empty |
| create duplicate | call `createBucket("a")` twice | second call returns duplicate-bucket error |
| create if missing create path | call `createBucketIfMissing("a")` on missing name | bucket is created once |
| create if missing existing path | call `createBucketIfMissing("a")` twice | second call returns existing bucket and catalog count stays one |
| drop existing | create `a`, drop `a`, commit, reopen | `getBucket("a")` returns `null` |
| drop missing | drop absent bucket | returns `false` and staged catalog root is unchanged |
| list order | create `b`, `a`, `c` | `listBuckets()` returns `a`, `b`, `c` |
| list filtering | catalog has marker plus user buckets | `listBuckets()` hides internal records |
| invalid empty name | any API with empty name | invalid-name error |
| invalid reserved prefix | any API with first byte `0x00` | invalid-name error |
| valid raw bytes | name contains invalid UTF-8 or embedded zero after byte 0 | operation is accepted |
| read-only guard | `createBucket`, `createBucketIfMissing`, or `dropBucket` in read tx | `ReadOnlyTransaction` |
| closed tx guard | use any bucket-management API after commit or rollback | `TransactionClosed` |
| reader snapshot create | reader begins, writer creates and commits bucket | old reader does not see bucket; new reader does |
| reader snapshot drop | reader begins, writer drops and commits bucket | old reader still sees bucket; new reader does not |
| writer read-your-writes create | writer creates bucket then calls `getBucket` and `listBuckets` | staged bucket is visible |
| writer read-your-writes drop | writer drops bucket then calls `getBucket` and `listBuckets` | dropped bucket is absent from staged view |
| rollback create | create bucket then rollback | bucket is absent after rollback |
| rollback drop | drop existing bucket then rollback | bucket remains visible after rollback |
| handle invalidation drop | use old handle after same-tx drop | invalid-state or transaction-lifecycle error |
| drop pending free | drop multi-page bucket | all reachable bucket pages recorded in `pending_free[new_txn_id]` |
| recreate after drop | drop `a`, recreate `a` in same write tx | new bucket is empty and uses a new root page |
| crash boundary create | inject failure before metadata switch during create commit | recovery selects old or new catalog according to M4 rules |
| crash boundary drop | inject failure before metadata switch during drop commit | recovery selects old or new catalog according to M4 rules |

Test names should stay behavior-oriented, for example
`bucket_create_if_missing_is_idempotent` and
`bucket_drop_preserves_older_reader_snapshot`.

## 10. M6 Handoff

M6 introduces overflow-page handling, deeper freelist behavior, and likely
statistics or introspection paths. This feature should hand off:

- one clear place to validate bucket names and catalog descriptors
- one clear bucket descriptor root-page source for future bucket statistics
- drop traversal hooks that can later include overflow chains without changing
  the public bucket-management API
- catalog iteration helpers that later stats APIs can reuse for bucket counts or
  bucket listings
- stable invalidation behavior for same-transaction bucket lifecycle changes

M5 should not add an approximate bucket count cache or bucket statistics side
table here. M6 can build those on top of the catalog and descriptor layer once
overflow and richer accounting rules exist.

## 11. Acceptance Criteria

This feature is complete for M5 when:

- `getBucket()`, `createBucket()`, `createBucketIfMissing()`, `dropBucket()`,
  and `listBuckets()` exist with the documented nullable and boolean behavior
- empty names and names starting with `0x00` are rejected consistently
- valid non-text raw-byte names remain supported
- mutating bucket-management APIs require a live write transaction
- read transactions keep stable bucket-catalog snapshots across concurrent
  writer commits
- write transactions see their own staged bucket creates and drops
- `createBucket()` rejects duplicates and `createBucketIfMissing()` is
  idempotent on an existing bucket
- `dropBucket()` removes exactly one existing bucket, invalidates same-tx
  handles, and records dropped pages as pending-free rather than reusable
- `listBuckets()` returns only user buckets in raw lexicographic byte order
- rollback removes staged create/drop catalog mutations completely
- reopen after successful commit preserves bucket existence and listing order
- failure injection around commit keeps bucket-management durability behavior
  consistent with the M4 metadata-switch rules
- tests cover API behavior, name validation, lifecycle guards, snapshot
  visibility, rollback, pending-free recording, and crash-boundary outcomes

## 12. Non-Goals

This feature does not implement:

- bucket-local KV mutation or lookup semantics beyond returning bucket handles
- cursor movement, range scans, or prefix scans
- nested buckets or secondary namespaces
- migration of legacy non-empty pre-M5 single-key-space roots
- compaction, vacuum, or bucket statistics APIs
- optimized mutation-while-iterating behavior
