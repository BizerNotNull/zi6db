# Single-Keyspace KV API Semantics

This document is the feature-level implementation plan for the M2
single-keyspace key-value API. It refines the API semantics from
[`m2-implementation-plan.md`](../m2-implementation-plan.md) and assumes the M1
storage lifecycle described in
[`m1-implementation-plan.md`](../../m1/m1-implementation-plan.md).

M2 exposes one unnamed ordered key space backed by the M0 B+Tree page format. It
does not expose public transactions yet; each successful mutation is published as
one internal auto-commit write unit.

## 1. Goals

- Provide concrete semantics for `get`, `exists`, `put`, and `delete`.
- Define key, value, allocator, and buffer ownership rules.
- Preserve M0/M1 lifecycle, metadata, read-only, and error contracts.
- Keep mutation planning separate from metadata publication so M3 can wrap the
  same behavior in public transactions.
- Define tests and acceptance criteria for the public KV surface.

## 2. Proposed Public Surface

Exact Zig module names may follow the M1 implementation, but the behavior should
match this shape:

```zig
pub fn get(db: *DB, allocator: Allocator, key: []const u8) !?[]u8;
pub fn exists(db: *DB, key: []const u8) !bool;
pub fn put(db: *DB, key: []const u8, value: []const u8) !void;
pub fn delete(db: *DB, key: []const u8) !bool;
```

The API operates only on the database's single unnamed key space. There are no
bucket names, table names, namespaces, cursors, range scans, or transaction
handles in this feature.

## 3. Key Semantics

- Keys are raw byte slices.
- Ordering is lexicographic over unsigned byte values.
- If equal bytes continue through the shorter slice, the shorter key sorts
  first.
- Duplicate keys are not allowed anywhere in the reachable tree.
- Empty keys are allowed unless implementation of the frozen page format proves
  they conflict with a hard invariant. If rejected, the public error and tests
  must document that decision before implementation proceeds.
- Key bytes are copied into B+Tree pages during `put`; the caller may reuse or
  free the input key buffer after the API call returns.
- `get`, `exists`, and `delete` borrow the caller's key only for the duration of
  the call.
- A key must fit inline in a leaf entry with its value and slot overhead. M2 does
  not support overflow keys.

## 4. Value Semantics

- Values are raw byte slices.
- Empty values are valid.
- `put` copies the value bytes into B+Tree pages before returning.
- `put` replaces the entire existing value when the key already exists.
- There is no append, compare-and-swap, duplicate-value, partial-update, or merge
  operation.
- A value must fit inline in one leaf entry with its key and slot overhead. M2
  must reject oversized entries deterministically.
- M2 must not truncate values or write overflow-page state reserved for later
  milestones.

## 5. `get`

`get` performs a point lookup in the selected metadata root.

Required behavior:

- Return `null` when the key is absent.
- Return `?[]u8` containing the full stored value when the key is present.
- Allocate the returned value with the allocator supplied by the caller.
- Return an independent copy; the returned buffer must not alias an internal page
  buffer, page cache, stack buffer, or temporary decode buffer.
- Leave database state unchanged.
- Be allowed on a read-only handle.

Failure behavior:

- If allocation fails, return the allocator's error and leave database state
  unchanged.
- If a reachable page is malformed during lookup, return `Corruption`.
- If storage I/O fails during lookup, return `IoError`.
- If the handle is closed or in an invalid lifecycle state, return
  `InvalidState`.

Caller responsibility:

- The caller owns the returned value buffer and must free it with the same
  allocator.

## 6. `exists`

`exists` performs the same point lookup as `get` but does not allocate or return
the stored value.

Required behavior:

- Return `true` when the key is present.
- Return `false` when the key is absent.
- Avoid value allocation.
- Leave database state unchanged.
- Be allowed on a read-only handle.

Implementation note:

- `exists` should share the same tree search and comparator as `get`.
- It may stop after locating the key and confirming that the leaf entry is
  structurally valid.

## 7. `put`

`put` inserts or replaces one key/value pair.

Required behavior:

- Insert a missing key.
- Replace the full value for an existing key.
- Treat replacement as a mutation even when the encoded value size stays the
  same.
- Copy key and value bytes before returning success.
- Publish the new root only through the inactive metadata slot.
- Increment `generation` and `txn_id` together after a successful mutation.
- Preserve all previously committed key/value pairs except the replaced key.
- Preserve lookup correctness after clean close and reopen.

Implementation flow:

1. Reject the call if the handle is read-only.
2. Acquire the M2 internal writer guard.
3. Capture the currently selected metadata snapshot.
4. Validate the key/value pair against inline leaf-entry size limits.
5. Search the B+Tree and record the root-to-leaf path.
6. Build a copy-on-write mutation result using newly allocated page ids.
7. Insert or replace the entry in the cloned leaf.
8. Split leaves and branches as needed, including root split.
9. Encode dirty pages and finalize checksums.
10. Publish through the internal auto-commit write unit.
11. Switch in-memory selected metadata only after metadata write and required
    sync succeed.
12. Release the writer guard.

No successful `put` may overwrite a currently visible root, branch, leaf, or
freelist page in place.

## 8. `delete`

`delete` removes one key if it exists.

Required behavior:

- Return `true` when an entry was removed.
- Return `false` when the key was already absent.
- Do not publish a new metadata generation for a missing key.
- Leave the file unchanged for a missing key except for incidental read-only
  state such as temporary buffers.
- Use copy-on-write for successful deletion.
- Preserve lookup correctness for all remaining keys after clean close and
  reopen.
- Be rejected on a read-only handle.

M2 delete is intentionally conservative:

- It may leave underfull leaves.
- It may leave empty non-root leaves.
- It does not need to merge, redistribute, rebalance, compact, or shrink the
  tree.
- It may keep a branch root after all logical entries are removed if search and
  checker logic deliberately accept that representation.
- It must update ancestor separators only when required to keep branch routing
  correct.

Implementation flow:

1. Reject the call if the handle is read-only.
2. Acquire the M2 internal writer guard.
3. Search for the key.
4. If missing, release the writer guard and return `false`.
5. Clone the root-to-leaf path.
6. Remove the entry from the cloned leaf.
7. Re-encode cloned ancestors with updated child page ids.
8. Preserve or adjust separators according to the branch routing invariant.
9. Encode dirty pages and finalize checksums.
10. Publish through the internal auto-commit write unit.
11. Return `true` only after publication succeeds.

## 9. Ownership And Lifetime

Input buffers:

- `key` and `value` parameters are borrowed for the duration of the call.
- `put` must copy the bytes needed for persistence before returning.
- Callers may mutate, reuse, or free input buffers after the call returns.

Returned buffers:

- `get` returns allocator-owned value memory.
- `get` must not expose internal page memory.
- Missing keys return `null` and allocate nothing.
- Allocation ownership must be documented in the public API comments.

Internal buffers:

- Decoded page views are internal and must not escape the API boundary.
- Promoted separator keys must be copied into mutation-owned memory before any
  temporary page decode buffer can be released.
- Staged dirty pages must not become visible to later reads unless metadata
  publication succeeds.

## 10. Read-Only Behavior

Read-only handles must support:

- `get`
- `exists`
- internal or public checker calls if present in the same milestone

Read-only handles must reject:

- `put`
- successful `delete`

The public error should be `ReadOnlyTransaction`, matching the M0 API rule. If
M1 exposes a differently named equivalent error, M2 may use that error only if
the mapping is documented in the implementation PR and tests.

Read-only rejection must happen before mutation planning, page allocation, or
metadata slot selection.

## 11. Error Mapping

Public errors should stay aligned with M0/M1 categories.

| Condition | Public error |
| --- | --- |
| malformed reachable tree page, bad checksum, invalid child range, duplicate key | `Corruption` |
| unsupported format version or required feature flag discovered on open | `IncompatibleVersion` |
| `put` or `delete` on a read-only handle | `ReadOnlyTransaction` |
| writer guard or unsupported locking conflict | `WriteConflict` or `LockConflict` |
| page read, page write, short I/O, flush, or sync failure | `IoError` |
| use after close, ambiguous failed-write state if reads are disabled | `InvalidState` |
| key/value pair cannot fit in one inline leaf entry | `EntryTooLarge` or the documented existing equivalent |
| allocator failure for `get` result or mutation staging | allocator error |

The oversized-entry error must be deterministic. It must not depend on partial
mutation progress or the current shape of unrelated pages.

If a write fails after any non-metadata page write, the in-memory selected
metadata must remain unchanged. If metadata write or sync may have partially
completed, the handle must enter the documented reopen-required state from the
M2 implementation plan before accepting later writes.

## 12. Lifecycle Rules

- `DB.create` initializes the M1 shape: header, metadata A, invalid metadata B,
  empty root leaf, and empty freelist.
- `DB.open` selects and validates metadata before any KV call can run.
- KV calls require an open handle.
- `close` releases storage resources and writer guards.
- `close` must not publish staged mutation state.
- `close` must return `InvalidState` if the implementation already tracks active
  operations or future transaction placeholders that make closure unsafe.
- No M2 API call may silently reopen a closed handle.
- No M2 API call may repair corruption as a side effect.

M2 does not expose snapshot semantics. If a later implementation internally
keeps old metadata snapshots for safety, that behavior must remain private until
M3 defines public transaction lifetimes.

## 13. Internal Implementation Boundaries

The API layer should be thin and deterministic:

- Validate lifecycle and read-only state.
- Map public errors.
- Enforce ownership rules.
- Delegate search and mutation to tree modules.
- Delegate metadata publication to the internal write unit.

Recommended internal seams:

- `tree.search` returns found/missing plus the leaf position.
- `tree.mutate_put` and `tree.mutate_delete` produce staged mutation results.
- `commit.simple_commit` owns page writes, sync ordering, metadata slot
  switching, and in-memory metadata publication.
- `format` codecs own page encoding, decoding, checksum finalization, and
  structural validation.

These seams are required so M3 can introduce read and write transactions without
changing the meaning of `get`, `exists`, `put`, or `delete`.

## 14. Tests

Automated tests should be behavior-driven and map back to this feature.

Core API tests:

- Empty database: `get` returns `null`, `exists` returns `false`, `delete`
  returns `false`.
- Single key: `put`, `get`, `exists`, `delete`, and second `delete`.
- Multiple keys: sequential, reverse, and random insert order.
- Binary keys: embedded `0x00`, high-bit bytes, prefix keys, and empty key if
  supported.
- Values: empty value, short value, long inline value, replacement with same,
  smaller, and larger encoded sizes.
- Oversized entry: maximum inline entry succeeds and too-large entry fails
  without changing existing data.

Persistence tests:

- Reopen after insert.
- Reopen after replacement.
- Reopen after delete.
- Reopen after leaf split.
- Reopen after root split.
- Reopen after mixed put/delete/replace workload.

Delete routing tests:

- Delete first key in a leaf.
- Delete last key in a leaf.
- Delete a key equal to a separator boundary.
- Delete the first key in a right subtree.
- Delete all keys from one non-root leaf.
- Delete all logical keys from a multi-level tree.
- Confirm all remaining keys are reachable and all deleted keys are absent.

Read-only tests:

- `get` and `exists` succeed on a read-only handle.
- `put` fails with the read-only error.
- `delete` fails with the read-only error.
- Failed read-only mutations do not allocate pages or publish metadata.

Failure-state tests:

- Allocation failure in `get` leaves state unchanged.
- Non-metadata write failure leaves old in-memory metadata selected.
- Metadata write or sync ambiguity moves the handle to the documented
  reopen-required policy.
- Missing-key delete does not increment `generation` or `txn_id`.

Checker-backed tests:

- Valid workloads pass the M2 checker.
- Unsorted leaf, duplicate key, invalid child range, wrong child page type,
  duplicate child pointer, invalid root type, and checksum failure are reported
  as corruption.

Randomized test:

- Use a fixed-seed operation stream.
- Compare `put`, `delete`, `get`, and `exists` with an in-memory sorted map.
- Periodically close and reopen.
- Run the checker after reopen or after each operation batch.

## 15. M3 Handoff

M2 should leave the KV semantics stable enough for M3 to add public
transactions as wrappers instead of redefining behavior.

Required handoff points:

- `get` and `exists` semantics become read-transaction point lookup semantics.
- `put` and `delete` mutation planning can be reused inside a future write
  transaction.
- The internal auto-commit write unit becomes the single-operation form of M3
  commit.
- Staged mutation results must remain separate from metadata publication.
- In-memory metadata switching must stay centralized in the commit layer.
- Returned value ownership remains allocator-owned unless M3 deliberately adds a
  separate borrowed-value API with transaction-bound lifetime.
- M3 may add snapshot isolation, rollback, and multi-operation atomicity without
  changing key ordering, duplicate-key, delete, or oversized-entry behavior.

M2 must not encode assumptions that every mutation is permanently a standalone
public operation. The standalone behavior is an API policy for M2, not a tree or
page-format limitation.

## 16. Non-Goals

This feature must not implement or specify:

- Public transaction objects.
- Multi-operation atomicity.
- Snapshot isolation.
- Rollback API.
- Buckets, tables, or named key spaces.
- Cursors, range scans, prefix scans, or iteration.
- Merge, redistribution, rebalance, or root collapse after delete.
- Overflow pages for large keys or values.
- Freelist reuse as a performance feature.
- Compaction, vacuum, backup, statistics, or production verify tooling.
- Crash-fault injection claims beyond the M2 clean-write ordering contract.
- Cross-process locking hardening beyond the M1 guard behavior.
- Absolute benchmark throughput targets.

## 17. Acceptance Checklist

This feature is complete when all of the following are true:

- `get`, `exists`, `put`, and `delete` are available for one unnamed key space.
- All API calls use the same unsigned-byte lexicographic comparator.
- `get` returns allocator-owned value copies and `null` for missing keys.
- `exists` performs point lookup without value allocation.
- `put` inserts missing keys and replaces existing values.
- `delete` removes existing keys, returns `false` for missing keys, and does not
  publish a new generation for missing keys.
- Caller-owned input buffers may be reused immediately after each API call.
- Empty values are supported.
- Empty keys are either supported or explicitly rejected with documented tests.
- Oversized inline entries fail cleanly without corrupting existing data.
- Read-only handles allow reads and reject mutations with the documented error.
- Successful mutations publish through metadata and increment `generation` and
  `txn_id` together.
- Failed mutations do not switch in-memory metadata or expose staged dirty pages.
- Clean close and reopen preserve inserted, replaced, and deleted state.
- Delete without rebalance preserves lookup correctness and checker validity.
- Tests cover empty DB, binary keys, replacement, deletion, split persistence,
  read-only behavior, failure states, checker failures, and fixed-seed randomized
  correctness.
- The implementation keeps mutation planning separate from metadata publication
  so M3 can add transactions without changing these API semantics.
