# Bucket-Local KV Operations

This document is the feature-level implementation plan for the M5 bucket-local
key-value API. It refines the bucket semantics from
[`m5-implementation-plan.md`](../m5-implementation-plan.md), reuses the M2
single-keyspace KV rules from
[`docs/design/m2/features/single-keyspace-kv-api.md`](../../m2/features/single-keyspace-kv-api.md),
and assumes the transaction and snapshot rules introduced by M3 and M4.

The goal is to turn the M2 B+Tree primitive into a bucket-scoped KV surface
without weakening copy-on-write isolation, metadata-root visibility, or the
stable raw-byte ordering rules already frozen by earlier milestones.

## 1. Goals

- Define concrete semantics for per-bucket `get`, `put`, `delete`, and
  `exists`.
- Preserve the M2 raw-byte key/value rules inside each bucket.
- Specify isolation between buckets so one bucket's mutations never affect
  another bucket's visible key space.
- Define write-transaction read-your-writes behavior for bucket-local reads.
- Define read-snapshot and write-snapshot rules for bucket-local operations.
- Provide a focused implementation order, tests, and acceptance criteria.

## 2. Scope

In scope:

- bucket-scoped point operations for `get`, `put`, `delete`, and `exists`
- semantics for missing keys, replacement, empty values, and key reuse
- bucket-local visibility rules inside read and write transactions
- isolation between catalog state and per-bucket user data
- snapshot pinning and read-your-writes behavior for bucket-local lookups
- tests for per-bucket correctness, isolation, persistence, and lifecycle

Out of scope:

- bucket creation, deletion, and listing API details beyond the assumptions this
  feature depends on
- cursor movement, range scans, and prefix scans
- nested buckets
- overflow-value behavior beyond preserving M2/M6 limits
- cross-process concurrency or lock policy

## 3. Proposed Surface

Exact Zig names may follow the implementation, but behavior should match this
shape:

```zig
pub fn get(bucket: *Bucket, allocator: Allocator, key: []const u8) !?[]u8;
pub fn exists(bucket: *Bucket, key: []const u8) !bool;
pub fn put(bucket: *Bucket, key: []const u8, value: []const u8) !void;
pub fn delete(bucket: *Bucket, key: []const u8) !bool;
```

These operations are always scoped to exactly one `Bucket` handle derived from a
single transaction. They do not address the catalog directly and they must never
accept a bucket name in place of a key.

## 4. Bucket-Local Data Model

Each user bucket owns one B+Tree rooted at the `bucket_root_page_id` stored in
its catalog descriptor.

Required rules:

- Bucket user keys are stored only in that bucket's B+Tree.
- Catalog keys remain bucket names and internal catalog records only.
- A bucket-local KV operation must start from the bucket root visible to the
  owning transaction, not from `metadata.root_page_id` directly.
- The same raw key bytes may exist in multiple buckets at the same time with
  different values.
- A lookup in one bucket must never search sibling buckets, the catalog tree, or
  another transaction's staged root.

Implementation note:

- The M2 search and mutation code should be reused as a tree primitive that
  accepts an explicit root page id and transaction context.
- The bucket layer is responsible for supplying the correct visible root for the
  current transaction and bucket handle.

## 5. Key And Value Semantics

Bucket-local KV operations inherit the M2 byte-level semantics unless M5
documents an explicit override.

Required rules:

- Keys are raw byte slices ordered lexicographically as unsigned bytes.
- Values are raw byte slices.
- Empty values are valid.
- Empty keys are valid unless the M2 implementation already documented and
  tested a hard rejection.
- Duplicate keys are not allowed within one bucket.
- The same key may exist once in bucket `a` and once in bucket `b` without
  conflict.
- `put` copies key and value bytes before success returns.
- `get` returns an allocator-owned copy and must not expose internal page
  memory.
- Oversized inline entries must fail deterministically with the documented
  existing error instead of silently truncating or spilling outside supported
  formats.

## 6. Operation Semantics

### 6.1 `get`

`get` performs a point lookup inside the bucket root visible to the bucket's
transaction.

Required behavior:

- Return `null` when the key is absent from this bucket.
- Return the full stored value when the key is present in this bucket.
- Allocate returned bytes with the caller-supplied allocator.
- Leave transaction state unchanged.
- Be allowed on read and write transactions, provided the bucket handle is still
  valid.

### 6.2 `exists`

`exists` performs the same bucket-local search as `get` without allocating a
value buffer.

Required behavior:

- Return `true` only when the key exists in this bucket.
- Return `false` when the key is absent from this bucket.
- Share the same search root, comparator, and corruption detection as `get`.

### 6.3 `put`

`put` inserts or replaces one key/value pair inside the selected bucket.

Required behavior:

- Insert a missing key into this bucket only.
- Replace the full value when the key already exists in this bucket.
- Require a bucket derived from an active write transaction.
- Stage mutations through transaction-private copy-on-write pages.
- Update the write transaction's visible bucket root so later bucket-local reads
  in the same transaction observe the staged value.
- Never mutate committed visible pages in place.

### 6.4 `delete`

`delete` removes one key from the selected bucket if it exists.

Required behavior:

- Return `true` when the key existed in this bucket and was removed.
- Return `false` when the key was already absent from this bucket.
- Require a bucket derived from an active write transaction.
- Leave other buckets unchanged.
- Avoid publishing any new metadata generation when the key is missing and the
  transaction later commits without other changes.

## 7. Isolation Rules

Bucket-local operations must preserve two independent isolation boundaries.

### 7.1 Bucket-To-Bucket Isolation

- A mutation in bucket `a` must not change the visible root, key set, or search
  result of bucket `b`.
- Replacing `k` in bucket `a` must not affect `get(b, k)`.
- Deleting `k` in bucket `a` must not create a tombstone interpretation in
  bucket `b`; bucket `b` simply keeps its own independent entry or absence.
- A bucket handle is bound to one descriptor/root pairing within one
  transaction; it must not be retargeted to another bucket after creation.

### 7.2 Transaction Isolation

- Read transactions observe only the metadata snapshot pinned at `beginRead()`.
- Write transactions build private staged bucket roots that remain invisible to
  other transactions until commit.
- A read transaction must never observe another transaction's uncommitted
  bucket-local `put` or `delete`.
- A write transaction must never consult committed shared-cache state after it
  already owns a private clone for the page being mutated.
- Commit publishes bucket-local mutations only through the normal M3/M4 metadata
  switch path.

## 8. Read-Your-Writes

M5 must provide read-your-writes inside one active write transaction.

Required behavior:

- After `put(bucket, k, v)` succeeds, a later `get(bucket, k)` in the same write
  transaction returns `v`.
- After `put(bucket, k, v)` succeeds, a later `exists(bucket, k)` in the same
  write transaction returns `true`.
- After `delete(bucket, k)` succeeds, a later `get(bucket, k)` in the same write
  transaction returns `null`.
- After `delete(bucket, k)` succeeds, a later `exists(bucket, k)` in the same
  write transaction returns `false`.
- Read-your-writes applies only within the same bucket handle lineage. It must
  not make staged writes visible to another transaction or another process.

Implementation requirements:

- Each write transaction should track the current staged root page id per touched
  bucket.
- Bucket-local reads in a write transaction must resolve through that staged root
  instead of the committed descriptor root from the base metadata snapshot.
- If one write transaction mutates multiple buckets, read-your-writes must be
  correct independently for each touched bucket.

## 9. Snapshot Rules

Bucket-local operations follow transaction snapshot rules, not ad hoc latest-read
behavior.

### 9.1 Read Transaction Snapshots

- A read transaction pins the catalog root and every bucket descriptor reachable
  from the selected metadata snapshot at transaction start.
- `get` and `exists` on a bucket from a read transaction must keep using that
  pinned bucket root for the lifetime of the transaction.
- If another transaction commits a replacement or delete for the same bucket and
  key, the already-open read transaction continues to see the old value or old
  absence.
- Reopening a new read transaction after the writer commit must observe the new
  committed bucket root.

### 9.2 Write Transaction Snapshots

- A write transaction starts from one pinned base metadata snapshot.
- Untouched buckets read from their committed descriptor roots in that base
  snapshot.
- Touched buckets read from their staged roots after the first successful
  mutation in the transaction.
- Rollback discards every staged bucket root and restores the base snapshot view
  because no metadata switch occurred.
- Commit atomically publishes the set of staged bucket roots together with the
  staged catalog root and freelist state.

### 9.3 Bucket Handle Stability

- A `Bucket` handle from a read transaction stays bound to the pinned descriptor
  and root until the transaction closes.
- A `Bucket` handle from a write transaction stays bound to the transaction's
  current staged state for that bucket.
- After transaction commit or rollback, all bucket handles become invalid and
  later KV calls must return the documented closed-transaction or invalid-state
  error.

## 10. Error And Lifecycle Rules

Bucket-local KV calls should preserve existing public error classes where
possible.

| Condition | Expected result |
| --- | --- |
| missing key | `get` returns `null`, `exists` returns `false`, `delete` returns `false` |
| mutation through read transaction bucket | `ReadOnlyTransaction` or documented equivalent |
| bucket handle used after transaction close | `TransactionClosed` or documented invalid-state equivalent |
| malformed reachable bucket page or descriptor-root mismatch | `Corruption` |
| unsupported format or feature requirement discovered during open | `IncompatibleVersion` |
| storage read/write/sync failure | `IoError` |
| key/value too large for inline entry limits | documented size-limit error |
| allocator failure for `get` result | allocator error |

Lifecycle rules:

- `get` and `exists` are allowed on valid bucket handles from read or write
  transactions.
- `put` and `delete` are allowed only on valid bucket handles from write
  transactions.
- No bucket-local operation may silently reopen a transaction or refresh a
  bucket's root from newer metadata.
- No bucket-local operation may bypass transaction close checks.

## 11. Implementation Plan

1. Reuse the M2 B+Tree point-search helper behind an interface that accepts an
   explicit bucket root and transaction read context.
2. Add a bucket-handle field or accessor that resolves the visible root page id
   for the current transaction state.
3. Route `get` and `exists` through the bucket-visible root instead of the
   database metadata root.
4. Reuse the M2 copy-on-write `put` and `delete` mutation planner inside
   `WriteTx`, scoped to one bucket root at a time.
5. After a successful bucket mutation, update the write transaction's staged
   root mapping for that bucket.
6. Ensure subsequent bucket-local reads in the same write transaction consult
   the staged root mapping first.
7. Ensure untouched buckets in the same write transaction still resolve through
   the base snapshot descriptors.
8. Add lifecycle guards so closed transactions, dropped buckets, and read-only
   mutation attempts fail before tree mutation starts.
9. Add tests for per-bucket isolation, read-your-writes, rollback, reopen, and
   concurrent reader snapshots.

## 12. Test Matrix

| Area | Scenario | Expected result |
| --- | --- | --- |
| missing key | `get`, `exists`, `delete` on empty bucket | `null`, `false`, `false` |
| insert in one bucket | put `k=v` in bucket `a` | `get(a, k)` returns `v` |
| same key in two buckets | put `k=v1` in `a`, `k=v2` in `b` | each bucket returns its own value |
| replace in one bucket | put `k=v1`, then `k=v2` in `a` | `get(a, k)` returns `v2` |
| delete existing | put then delete `k` in `a` | delete returns `true`, later `get` is `null` |
| delete missing | delete absent `k` in `a` | returns `false`, no other bucket changes |
| empty value | put `k=""` | `get` returns empty byte slice |
| binary key | embedded `0x00` or high-bit key in `a` | lookup succeeds by raw-byte order |
| empty key | put and get empty key if supported | behavior matches documented M2 rule |
| size limit | oversized key/value entry in `a` | documented size-limit error, old data unchanged |
| bucket isolation read | key exists only in `a` | `get(b, k)` returns `null` |
| bucket isolation delete | delete `k` in `a` while `b` has `k` | `b` still returns its own value |
| read-your-writes put | write tx puts `k=v` then gets `k` | staged value is visible before commit |
| read-your-writes delete | write tx deletes `k` then gets `k` | staged absence is visible before commit |
| multi-bucket writer | write tx mutates `a` and `b` | reads from each bucket see the correct staged root |
| reader snapshot old value | reader starts, writer replaces `k`, writer commits | old reader sees old value, new reader sees new value |
| reader snapshot old absence | reader starts before writer inserts `k` | old reader sees missing, new reader sees present after commit |
| rollback | write tx puts or deletes keys then rolls back | reopened read shows pre-transaction state |
| commit persistence | write tx mutates bucket then commits and reopen | new value/absence persists after reopen |
| read-only guard | `put` or `delete` through read tx bucket | read-only error |
| transaction close | use bucket after commit or rollback | closed-transaction or invalid-state error |
| corruption surfacing | malformed reachable bucket page | `Corruption` |
| unaffected untouched bucket | writer mutates `a`, commits | `b` contents remain unchanged before and after reopen |

Recommended test group names:

- `bucket_kv_basic`
- `bucket_kv_isolation`
- `bucket_kv_read_your_writes`
- `bucket_kv_snapshot_rules`
- `bucket_kv_persistence`

## 13. Acceptance Checklist

This feature is complete when all of the following are true:

- Per-bucket `get`, `put`, `delete`, and `exists` semantics are documented and
  implemented.
- Bucket-local operations reuse M2 raw-byte key ordering and value semantics.
- The same key bytes can exist in different buckets without conflict.
- A bucket-local lookup never escapes its bucket root into another bucket or the
  catalog tree.
- `put` and `delete` require a write transaction bucket and stage copy-on-write
  mutations privately.
- Write transactions provide read-your-writes for later bucket-local `get` and
  `exists` calls in the same transaction.
- Read transactions keep a stable bucket snapshot for their full lifetime,
  regardless of later writer commits.
- Rollback discards staged bucket-local changes completely.
- Commit publishes bucket-local changes only at the metadata visibility
  boundary.
- Missing-key behavior is deterministic: `get -> null`, `exists -> false`,
  `delete -> false`.
- Bucket-local lifecycle and read-only misuse return documented errors instead of
  silently rebasing or mutating.
- Tests cover basic CRUD, replacement, empty values, binary keys, cross-bucket
  isolation, read-your-writes, snapshot stability, rollback, commit persistence,
  and closed-transaction misuse.
