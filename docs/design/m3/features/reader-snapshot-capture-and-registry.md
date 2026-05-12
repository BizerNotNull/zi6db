# Reader Snapshot Capture And Registry

## 1. Purpose

This document turns the M3 requirement for stable read snapshots into a concrete
implementation plan for a single-process reader registry.

The feature must ensure that:

- `beginRead()` captures one committed metadata snapshot and never rebases.
- reader registration is atomic with metadata capture
- `max_read_transactions` is enforced without leaking registry state
- snapshot reads never observe uncommitted writer state
- writers can compute `oldest_reader_txn_id` conservatively for pending-free
  promotion
- the M3 implementation leaves clear seams for M6 statistics and M7
  multi-process reader tracking

This plan follows:

- [docs/design/m3/m3-implementation-plan.md](/D:/代码D/zi6db/docs/design/m3/m3-implementation-plan.md)
- [docs/design/m0/transaction-recovery.md](/D:/代码D/zi6db/docs/design/m0/transaction-recovery.md)
- [AGENTS.md](/D:/代码D/zi6db/AGENTS.md)

## 2. Scope

This feature covers only in-process read snapshot capture and reader registry
behavior for M3.

It does not define:

- cross-process reader tables
- OS lock protocols
- crash recovery changes
- new on-disk metadata or freelist format rules
- bucket, cursor, or overflow-page APIs

## 3. Required Invariants

The implementation must preserve these invariants:

- a read transaction pins the currently selected committed metadata snapshot at
  begin time
- metadata capture and reader registration happen inside one critical section
- a reader slot is removed exactly once on transaction close
- multiple readers may share the same `snapshot_txn_id`, but their registry
  entries remain distinct
- `oldest_reader_txn_id` is derived from active registry entries only
- a writer may not reuse pending-free pages unless the reader registry proves
  that no older reader can still reach them
- registry cleanup after read errors must still happen even if later read
  operations return `Corruption` or `IoError`

## 4. Data Model

### 4.1 Read Transaction Snapshot

Each `ReadTx` should store an immutable snapshot record copied from the selected
in-memory metadata:

```zig
const ReadSnapshot = struct {
    selected_slot: MetadataSlot,
    generation: u64,
    txn_id: u64,
    root_page_id: u64,
    freelist_page_id: u64,
    page_count: u64,
};
```

The transaction also stores:

- `db: *DB`
- `state: TransactionState`
- `snapshot: ReadSnapshot`
- `reader_token: ReaderRegistryToken`

The snapshot record should be copied, not borrowed, so later in-memory metadata
publication cannot affect an already active reader.

### 4.2 DB Reader Registry

`DB` should own a small in-memory registry:

```zig
const ReaderRegistryEntry = struct {
    token: u64,
    snapshot_txn_id: u64,
};

const ReaderRegistry = struct {
    next_token: u64,
    active_count: u32,
    entries: std.ArrayListUnmanaged(ReaderRegistryEntry),
};
```

Required properties:

- `token` uniquely identifies one active reader entry
- `snapshot_txn_id` is the pinned visibility marker used for reuse decisions
- `active_count` supports fast limit checks
- the registry format stays process-local in M3 and does not become a persisted
  file structure

M3 does not need a complex heap or tree here. A compact vector plus linear scan
is acceptable unless profiling later proves otherwise.

## 5. Locking And Atomic Capture

### 5.1 Shared Critical Section

`beginRead()` must use one DB-local critical section that protects both:

- reading the currently selected in-memory metadata
- publishing the new reader entry

The lock may be a dedicated mutex or a combined metadata/reader-registry mutex.
The exact primitive is an implementation detail. The observable rule is that no
writer may sample reader state for pending-free promotion between metadata
capture and registry insertion.

### 5.2 Begin Read Flow

`beginRead()` should run in this order:

1. Reject begin if the database is closing or closed.
2. Acquire the metadata/reader-registry lock.
3. Check `active_count` against `max_read_transactions`.
4. Copy the selected metadata fields into a local `ReadSnapshot`.
5. Allocate a unique registry token.
6. Insert `{ token, snapshot_txn_id = snapshot.txn_id }` into the registry.
7. Increment `active_count`.
8. Initialize the `ReadTx` with the copied snapshot and token.
9. Release the metadata/reader-registry lock.
10. Return the active read transaction.

Failure handling:

- if the limit check fails, return the lifecycle error before any registry
  mutation
- if registry insertion fails, do not return a partially registered reader
- if transaction object construction fails after insertion, remove the inserted
  entry before releasing the lock or before returning the error

### 5.3 Close Read Flow

Read transaction cleanup should run in this order:

1. transition the transaction to a closed state
2. acquire the metadata/reader-registry lock
3. remove the entry matching `reader_token`
4. decrement `active_count`
5. release the lock
6. discard the transaction snapshot and token locally

Removal must be by token, not by `snapshot_txn_id`, because multiple readers
may pin the same committed state concurrently.

## 6. Metadata Pinning Rules

Metadata pinning is the contract that makes snapshot reads correct.

Rules:

- read transactions copy `selected_slot`, `generation`, `txn_id`,
  `root_page_id`, `freelist_page_id`, and `page_count` at begin time
- these fields remain immutable for the transaction lifetime
- read paths must traverse only from `snapshot.root_page_id`
- any bounds checks use `snapshot.page_count`, not the current DB page count
- any freelist-aware validation or stats path uses `snapshot.freelist_page_id`
- a later writer commit may publish newer metadata in memory, but active readers
  continue to use their own copied snapshot

This means M3 should treat the pinned snapshot as transaction-owned metadata,
not as a pointer to mutable DB-global state.

## 7. Snapshot Read Behavior

Snapshot reads must obey the following rules:

- never consult writer-private dirty pages
- never read the current DB root or freelist after the transaction begins
- reject page ids outside the pinned `page_count`
- return caller-owned values rather than borrowed storage slices
- treat corruption relative to the pinned snapshot as a reader-visible error

Observable behavior:

- a reader that begins before a writer commits continues to see the old value
- a reader that begins after that commit sees the new value
- an active writer may mutate cloned pages privately without affecting any
  existing reader

## 8. `max_read_transactions`

M3 should enforce `OpenOptions.max_read_transactions` as a hard cap on active
read transactions in one process.

Rules:

- the check happens while holding the metadata/reader-registry lock
- the limit is evaluated before the reader is returned
- a failed begin must not consume a token or leave an entry behind
- closed readers release capacity immediately

Suggested error behavior:

- return `InvalidState` if the current API keeps one lifecycle-style error for
  this condition
- alternatively introduce a dedicated limit error later, but M3 should document
  one consistent outcome and test for it

## 9. `oldest_reader_txn_id`

`oldest_reader_txn_id` is the registry-derived cutoff used by writers to decide
whether older `pending_free` groups may move into `reusable`.

Recommended API shape:

```zig
pub fn oldestReaderTxnId(db: *DB) ?u64;
```

Behavior:

- return `null` when no readers are active
- otherwise return the minimum `snapshot_txn_id` among active registry entries
- compute the value while holding the metadata/reader-registry lock

This function is process-local in M3. That is sufficient because M3 does not
yet allow multi-process reuse decisions.

Writer usage:

1. acquire the metadata/reader-registry view needed for commit-time promotion
2. sample `oldest_reader_txn_id`
3. promote only `pending_free[X]` groups where no active reader has
   `snapshot_txn_id < X`
4. preserve all not-yet-eligible groups exactly as pending

M3 must remain conservative. If registry state is uncertain, promotion should
not happen.

## 10. Implementation Steps

Implement this feature in the following order:

1. Add `ReadSnapshot`, `ReaderRegistryEntry`, and `ReaderRegistryToken` types.
2. Add DB-local metadata/reader-registry locking.
3. Add registry insertion, removal, and minimum-txn scan helpers.
4. Update `beginRead()` to capture metadata and register the reader atomically.
5. Update `rollback(ReadTx)` or equivalent close logic to remove the reader by
   token.
6. Route read traversal and page bounds checks through the pinned snapshot
   fields.
7. Add the `max_read_transactions` guard inside the begin critical section.
8. Expose or internally use `oldest_reader_txn_id` for writer-side
   pending-free promotion.
9. Add tests for lifecycle, limit, snapshot isolation, and cleanup on error.

## 11. Test Plan

Minimum tests for this feature:

| Area | Scenario | Expected result |
| --- | --- | --- |
| Atomic capture | Reader begins while writer is preparing a commit | Reader is either fully absent or fully registered against exactly one copied snapshot |
| Metadata pinning | Reader starts before writer commit | Reader keeps old `txn_id`, root, freelist, and page count |
| Snapshot reads | Reader before commit, second reader after commit | First reader sees old value, second reader sees new value |
| Dirty isolation | Writer updates key before commit | Existing reader never observes uncommitted value |
| Limit enforcement | Active readers equal `max_read_transactions` | Next `beginRead()` fails and registry count does not change |
| Token cleanup | Two readers share the same `snapshot_txn_id`, one closes | Only that reader is removed, the other remains active |
| Error cleanup | Reader operation returns `Corruption` or `IoError`, then closes | Registry entry is still removed exactly once |
| Oldest reader | Readers pin `txn_id` 10, 12, and 12 | `oldest_reader_txn_id` reports `10` |
| Empty oldest reader | No active readers | `oldest_reader_txn_id` returns `null` |
| Pending-free safety | Oldest reader is older than a pending-free group cutoff | Writer keeps that group pending, not reusable |

The tests should also assert negative conditions:

- no leaked reader entries after failed `beginRead()`
- no double removal on repeated close paths
- no registry mutation by ordinary read operations

## 12. M6 Handoff

M6 statistics already plan to expose:

- `active_read_transactions`
- `oldest_reader_txn_id`
- snapshot-stable `statsTx(tx)`

This M3 feature should hand off the following seams:

- a reliable active reader count from the registry
- a reliable `oldest_reader_txn_id` query over active entries
- transaction-local snapshot fields that M6 stats can read without consulting
  mutable DB-global metadata

M3 should not add a second parallel reader-count mechanism. M6 should consume
the same registry that M3 uses for reuse safety.

## 13. M7 Handoff

M7 extends reader tracking from process-local state to a multi-process
coordination protocol.

This M3 design should hand off:

- the semantic meaning of one reader entry: one active snapshot pinned to one
  `txn_id`
- removal by stable reader identity, not by bare `txn_id`
- conservative `oldest_reader_txn_id` semantics
- the rule that uncertain reader liveness blocks reuse rather than allowing it

M7 may replace the storage location of reader entries with a lock-protected
table or sidecar mechanism, but it should preserve the same logical contract:

- atomic snapshot publication for a new reader
- exact cleanup of one reader slot
- conservative minimum `txn_id` sampling before pending-free promotion

## 14. Acceptance Criteria

This feature is complete for M3 when:

- `beginRead()` copies the selected committed metadata into transaction-owned
  snapshot fields
- reader registration is atomic with metadata capture
- registry entries are keyed by a stable token and removed exactly once
- `max_read_transactions` is enforced without leaking capacity
- snapshot reads use pinned root, freelist, and page count values only
- active readers never observe uncommitted writer state
- `oldest_reader_txn_id` returns the minimum active snapshot `txn_id` or `null`
- writer-side pending-free promotion can use the registry conservatively
- the implementation exposes the same reader facts needed later by M6 stats and
  M7 multi-process tracking
- tests cover pinning, isolation, cleanup, limit handling, and oldest-reader
  computation
