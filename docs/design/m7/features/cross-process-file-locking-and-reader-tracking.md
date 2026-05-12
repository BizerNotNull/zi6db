# Cross-Process File Locking And Reader Tracking

This feature plan makes the M7 locking hardening work concrete for
cross-process file locking, reader tracking, stale-reader handling, and safe
freelist cutoff calculation. It is derived from
`docs/design/m7/m7-implementation-plan.md`,
`docs/design/m6/m6-implementation-plan.md`, and the repository rules in
`AGENTS.md`.

The goal is to replace the M6 process-local reader cutoff with a real
cross-process protocol that preserves the same logical concurrency model:

- many concurrent readers
- at most one writer-capable owner
- conservative page reuse based on the oldest visible reader snapshot
- no weakening of M0 metadata-switch, recovery, or freelist safety rules

## 1. Scope

In scope:

- cross-process lock mapping for Linux, macOS, and Windows
- lock byte/range protocol shared by all supported platforms
- writer-capable open behavior and read-only open behavior
- per-reader slot publication for `generation` and `txn_id`
- stale-reader detection and cleanup rules
- lock lifecycle across `open()`, begin read, begin write, commit, rollback,
  and `close()`
- typed lock-conflict and stale-reader failure paths
- deterministic multi-process tests
- acceptance criteria for M7 implementation and CI

Out of scope:

- changing the on-disk format or metadata layout
- storing dynamic reader state in the immutable header or metadata pages
- best-effort repair of broken lock state
- non-cooperating external writers that ignore the zi6db lock protocol
- background lock daemons or lease servers
- compaction, backup, or verify beyond the seams they depend on

## 2. Design Intent

M7 should turn the M6 `oldest_active_reader_txn_id` seam into a platform-backed,
cross-process protocol with these properties:

1. A writer-capable open proves exclusive writer ownership before it can publish
   metadata.
2. A read transaction publishes a conservative snapshot record that other
   cooperating writers can inspect.
3. Pending-free promotion uses the cross-process reader table rather than a
   process-local registry.
4. Stale reader state may delay reuse, but it must never allow unsafe early
   reuse.
5. All dynamic liveness decisions must be tied to OS lock ownership or another
   platform-proven liveness proof, not timeout-only heuristics.

The lock protocol is a runtime contract, not a format field. It may reserve
well-known byte ranges in the database file for OS locks, but it must not store
variable reader state in those bytes and must not require a format bump.

## 3. Lock Model And Byte-Range Layout

M7 should define one logical lock layout that maps cleanly to both POSIX
byte-range locks and Windows byte-range locks.

Recommended logical ranges:

| Range | Purpose | Lock Mode |
| --- | --- | --- |
| `open_guard` | shared lifecycle guard for cooperating opens and close sequencing | shared for readers and writers |
| `writer_guard` | exclusive writer-capable ownership and metadata publication gate | exclusive for one writer-capable process |
| `reader_slot[i]` | ownership proof for reader slot `i` | exclusive per active slot owner |
| `reader_scan_guard` | optional short critical section for table scan/update if the implementation needs one | short exclusive or shared/exclusive wrapper |

Recommended protocol rules:

- `open_guard` is acquired first by every cooperating open.
- `writer_guard` is acquired only by a writer-capable open and remains owned
  until `close()`.
- each active read transaction owns exactly one `reader_slot[i]` lock for its
  published slot
- slot locks are independent so multiple readers can coexist
- writers never clear another process's slot; they only inspect slot state and
  OS lock ownership
- metadata publication requires the writer-capable process to still own
  `writer_guard`

The implementation may choose exact byte offsets later, but the layout should
reserve enough fixed slot space to avoid format-visible migration. If the
initial slot count is bounded, exhaustion must fail safely with a typed error
such as `LockConflict` or `ReaderTableFull`; it must not silently fall back to
process-local tracking.

## 4. POSIX And Windows Lock Mapping

## 4.1 POSIX Mapping

Linux and macOS should use `fcntl` advisory byte-range locks over the shared
logical ranges.

Required POSIX rules:

- use `fcntl` consistently for all correctness-critical database locks
- do not mix `flock` with `fcntl` for the same database path
- treat locks as advisory and document that protection covers only cooperating
  zi6db processes
- acquire shared locks for `open_guard` on read-only and writer-capable opens
- acquire an exclusive lock on `writer_guard` for writer-capable opens
- acquire an exclusive lock on exactly one `reader_slot[i]` per active reader
- release locks automatically on process death or descriptor close, but do not
  rely on implicit release for normal cleanup paths

macOS should initially follow the same `fcntl` mapping as Linux. If CI reveals
platform-specific behavior differences, the implementation may add a wrapper
layer without changing the logical lock protocol.

## 4.2 Windows Mapping

Windows should use `LockFileEx` and `UnlockFileEx` over the same logical ranges.

Required Windows rules:

- `open_guard`, `writer_guard`, and `reader_slot[i]` use the same logical byte
  layout as POSIX
- shared reader-compatible opens and exclusive writer ownership are enforced
  through `LockFileEx`, not process-local flags
- file open sharing flags must allow the intended shared readers plus read-only
  maintenance operations such as verify or backup, while denying conflicting
  writer access consistently with byte-range locks
- lock failures must map to `LockConflict` rather than generic `IoError`
- the implementation must document which sharing flags are used for
  writer-capable opens, read-only opens, and maintenance opens

Recommended Windows open policy:

- read-only open allows `FILE_SHARE_READ` and any additional sharing required by
  zi6db maintenance operations
- writer-capable open denies incompatible writer access and relies on
  `writer_guard` for exact concurrency semantics
- maintenance paths such as read-only verify should behave like read-only opens
  unless a stronger lock mode is explicitly requested

## 5. Reader Table Design

M7 needs a durable-in-memory, cross-process-visible reader publication model
without changing M0/M6 on-disk semantics.

Recommended reader slot contents:

- `slot_generation`: monotonic local generation for the slot record
- `process_id`: OS process identifier
- `process_start_identity`: extra identity material so reused PIDs alone are
  not trusted
- `snapshot_generation`: selected metadata generation seen by the reader
- `snapshot_txn_id`: selected metadata transaction ID seen by the reader
- `state`: empty, publishing, active, or clearing

Design requirements:

- a slot becomes authoritative for reuse decisions only after the reader has
  fully published `snapshot_generation` and `snapshot_txn_id`
- slot publication must be ordered so scanners cannot mistake partial bytes for
  an active reader
- slot state changes must be paired with the owning `reader_slot[i]` OS lock
- empty slots must decode deterministically
- slot data is runtime metadata only; it must not participate in M0 recovery
  selection

Recommended publication sequence for a read transaction:

1. confirm the process already holds `open_guard`
2. find a free reader slot by probing slot lock ownership
3. acquire exclusive ownership of `reader_slot[i]`
4. write a `publishing` slot record with fresh slot generation and process
   identity
5. read and pin the selected metadata snapshot
6. write `snapshot_generation` and `snapshot_txn_id`
7. mark the slot `active`
8. expose the read transaction to the caller

Recommended cleanup sequence:

1. mark the slot `clearing` or zero it in one bounded write
2. flush ordering only as needed for intra-process visibility; this state is not
   crash-recovered data
3. release `reader_slot[i]`
4. forget the slot from process-local transaction state

If a read transaction fails before step 7, the implementation must release the
slot lock and leave the slot empty or clearly non-active.

## 6. Stale Reader Handling

Stale reader handling must remain conservative.

Required rules:

- timeout alone is never enough to reclaim a slot
- a slot may be cleaned only after the implementation proves the owning process
  no longer holds the corresponding OS slot lock
- PID reuse alone is never enough to declare the slot live or dead
- if liveness proof is ambiguous, the slot remains logically active for reuse
  decisions
- stale slots may delay `pending_free` promotion, but must not allow early
  promotion

Recommended stale-reader evaluation order:

1. inspect slot bytes
2. attempt the platform-specific non-destructive check for whether the
   `reader_slot[i]` lock is still owned
3. if the slot lock is still owned, treat the slot as live
4. if the slot lock is free, validate whether the slot record is stale and can
   be cleared
5. clear the slot only while holding the slot lock or another protocol step
   that prevents concurrent republishing races

Recommended stale states:

- `orphaned_slot`: slot bytes claim an active reader but the OS slot lock is no
  longer owned
- `partial_publish`: slot bytes are present but not in the committed `active`
  state
- `identity_mismatch`: PID matches a current process but the extra process
  identity does not

All of these states must resolve conservatively toward "block reuse until
proven safe."

## 7. Oldest Reader Calculation

M7 should replace the M6 process-local cutoff provider with a reader-table scan
that computes:

- `oldest_reader_txn_id`
- optionally `oldest_reader_generation` for diagnostics

Required scan rules:

- only slots with valid `active` state and proven current lock ownership count
  as live readers
- empty, clearing, or orphaned slots may be ignored only after the protocol
  proves they are not live
- if any slot cannot be classified safely, the calculation must fail closed and
  block promotion rather than guess
- pending-free promotion continues to use the M6 rule: group `X` is promotable
  only when no live reader has `snapshot_txn_id < X`

Integration rules:

- the allocator and freelist promotion logic should depend only on a provider
  interface like `oldest_reader_txn_id()`
- M7 should swap the provider implementation, not redesign M6 pending-group
  semantics
- if multi-process reader tracking initialization fails, writer-capable open in
  `lock_mode = .multi_process` must fail safely

## 8. Lock Lifecycle

M7 should make lock ownership explicit across the full database lifecycle.

## 8.1 `open()`

Read-only open:

1. open the file with read-only compatible sharing flags
2. acquire shared `open_guard`
3. validate header and selected metadata through existing recovery rules
4. publish the `DB` handle without writer capability

Writer-capable open:

1. open the file with writer-capable flags and documented sharing behavior
2. acquire shared `open_guard`
3. acquire exclusive `writer_guard`
4. initialize or validate the reader table protocol
5. validate header and selected metadata
6. publish the `DB` handle with writer capability

Failure rules:

- if any lock step fails, close the file handle and release partial locks
- if reader-table validation fails, release `writer_guard` and `open_guard`
- open conflicts fail fast by default and return `LockConflict`

## 8.2 Begin Read Transaction

- requires an open `DB`
- requires read-only compatibility but not writer ownership
- allocates and publishes one reader slot
- pins selected metadata only after slot publication ordering is safe
- on failure, releases any partially acquired slot lock and does not leak slot
  state

## 8.3 Begin Write Transaction

- requires a writer-capable `DB`
- confirms the process still owns `writer_guard`
- does not require any additional cross-process writer lock
- may sample the reader table to prepare conservative promotion state

If writer ownership has been lost or cannot be confirmed, begin-write must fail
with `InvalidState` or `LockConflict` rather than proceeding toward commit.

## 8.4 Commit And Rollback

Commit rules:

- the writer must still own `writer_guard` before metadata publication begins
- any reader-table sampling used for freelist promotion must happen under the
  documented scan protocol
- a failed commit after metadata-write start must not release or advance
  cross-process reader-cutoff assumptions as if the new metadata definitely won
- ambiguous commit results still follow M0: reopen and recovery are
  authoritative

Rollback rules:

- rollback releases no database-open locks beyond transaction-local reader slots
- rollback of a read transaction clears its slot and unlocks `reader_slot[i]`
- rollback of a write transaction preserves `writer_guard` for the owning `DB`
  until `close()`

## 8.5 `close()`

Required close behavior:

- `close()` fails with `InvalidState` if any read or write transactions remain
  active
- successful close releases reader slots first, then `writer_guard` if owned,
  then `open_guard`, then the file handle
- close-failure paths must not clear locks owned by other processes
- all release steps must be exception-safe and idempotent for local cleanup

## 9. Error Model And Diagnostics

Locking and reader tracking should preserve the public error model while adding
better context.

Required public categories:

- `LockConflict` for incompatible open or lock acquisition failure
- `WriteConflict` only when the existing public API distinguishes write-tx
  conflicts from open conflicts
- `InvalidState` for lifecycle misuse such as closing with active transactions
  or operating after writer ownership is lost
- `Corruption` if reader-table bytes are malformed in a way that violates the
  runtime protocol and safe classification is impossible
- `IoError` only for true I/O failures unrelated to a lock conflict

Diagnostic context should include:

- operation: `open`, `beginRead`, `beginWrite`, `commit`, `rollback`, `close`
- path when safe
- lock range or slot index when relevant
- process identity and snapshot `txn_id` only in bounded debug output
- whether the failure occurred during acquisition, publication, scan, cleanup,
  or verification

## 10. Tests

M7 should add deterministic multi-process tests with explicit IPC barriers
instead of timing sleeps.

## 10.1 Lock Mapping Tests

| Area | Scenario | Expected Result |
| --- | --- | --- |
| writer open | two writer-capable opens race | exactly one succeeds; the other returns `LockConflict` |
| shared open | read-only open while writer-capable open is active | succeeds when the documented lock model allows it |
| writer tx | second process attempts write transaction while writer-capable owner exists | fails safely with `LockConflict` or `WriteConflict` |
| POSIX mapping | cooperating processes use `fcntl` ranges | conflicting writer ownership is blocked; compatible readers coexist |
| Windows mapping | conflicting opens under sharing flags and byte-range locks | conflicts are denied with `LockConflict`, not generic I/O |
| maintenance open | read-only verify or backup while writer-capable open exists | succeeds or fails exactly as documented by lock mode |

## 10.2 Reader Table Tests

| Area | Scenario | Expected Result |
| --- | --- | --- |
| publish | read transaction publishes slot then reads snapshot | slot records selected `generation` and `txn_id` before becoming active |
| cleanup | read transaction closes normally | slot is cleared and lock released |
| crash cleanup | process dies with active reader | OS slot lock is released; stale slot is cleaned only after liveness proof |
| slot exhaustion | all reader slots are active | next read begin fails safely with typed error |
| partial publish | failure between slot acquisition and activation | slot does not become authoritative and is cleaned safely |
| PID reuse defense | stale slot PID is reused by another process | extra identity prevents false live/dead classification |

## 10.3 Stale Reader And Reuse Tests

| Area | Scenario | Expected Result |
| --- | --- | --- |
| conservative stale handling | stale-looking slot without liveness proof | pending-free promotion stays blocked |
| orphan cleanup | active-looking slot with no owned OS slot lock | slot can be reclaimed safely |
| oldest reader scan | multiple active readers across processes | minimum `snapshot_txn_id` is reported correctly |
| equality rule | oldest live reader has `snapshot_txn_id == freeing_txn_id` | pending group is promotable |
| older reader block | one live reader has `snapshot_txn_id < freeing_txn_id` | pending group remains blocked |
| writer crash | writer dies before releasing `writer_guard` | OS releases writer lock; next writer-capable open can proceed |

## 10.4 Lifecycle And Failure Tests

| Area | Scenario | Expected Result |
| --- | --- | --- |
| open failure cleanup | conflict during `open()` after partial acquisition | file handle and all partial locks are released |
| commit guard | writer loses `writer_guard` before metadata publish | commit fails safely and does not publish metadata |
| ambiguous commit | failure after metadata write start | reader-cutoff assumptions are not advanced; reopen is authoritative |
| close guard | `close()` with active transactions | returns `InvalidState` and keeps locks owned by live transactions |
| rollback cleanup | read rollback after slot publish | slot and slot lock are released exactly once |
| multi-path cleanup | open/read/commit failure combinations | no path leaks a slot or clears another process's slot |

Suggested behavior-oriented test names:

- `writer_capable_open_conflicts_across_processes`
- `reader_slot_is_not_counted_until_publication_reaches_active_state`
- `stale_reader_without_lock_ownership_blocks_reuse_until_cleaned`
- `commit_fails_if_writer_guard_is_lost_before_metadata_publication`

## 11. Execution Notes

Recommended implementation order:

1. define the logical lock ranges and the cross-platform lock abstraction
2. implement writer-capable open and read-only open around `open_guard` and
   `writer_guard`
3. add the reader slot table layout and transaction-local slot ownership state
4. implement reader publication and cleanup for read transactions
5. replace the M6 process-local cutoff provider with cross-process table scans
6. add stale-reader detection and conservative cleanup
7. wire lock and slot diagnostics into `open`, transaction, commit, and close
   paths
8. finish deterministic multi-process tests on Linux, macOS, and Windows

The feature should land only after the M6 allocator can consume the new cutoff
provider without changing pending-group semantics.

## 12. Acceptance

This feature is accepted when:

- Linux and macOS use `fcntl` advisory byte-range locks over a documented
  zi6db lock layout
- Windows uses `LockFileEx` and `UnlockFileEx` over the same logical lock
  ranges with documented file-sharing behavior
- writer-capable open proves exclusive writer ownership and conflicting writer
  opens fail with `LockConflict`
- read-only opens behave consistently with the documented shared-open model
- each active read transaction publishes a reader slot containing the pinned
  `generation` and `txn_id`
- `oldest_reader_txn_id` is computed from cooperating processes, not only the
  current process
- stale reader handling is conservative: stale state may delay reuse, but
  cannot make `pending_free` pages reusable early
- slot cleanup requires proof that the owning OS slot lock is no longer held
- PID reuse alone is never trusted for stale-reader classification
- writer metadata publication cannot proceed after writer ownership is lost
- lock acquisition, publication, rollback, commit failure, and close paths do
  not leak locks or clear another process's slot
- lock conflicts are reported as typed lock errors rather than generic
  `IoError`
- deterministic tests cover POSIX lock behavior, Windows lock behavior, reader
  publication, stale-reader cleanup, lifecycle cleanup, and conservative
  pending-free promotion
- the implementation preserves M0 commit ordering and M6 pending-free safety
  while replacing the process-local reader cutoff with a cross-process protocol
