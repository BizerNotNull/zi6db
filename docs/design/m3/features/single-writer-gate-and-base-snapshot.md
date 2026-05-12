# M3 Feature Plan: Single Writer Gate And Base Snapshot

This plan defines the concrete M3 implementation work for enforcing one active
writer and capturing a stable write base snapshot before any copy-on-write
mutation begins. It refines `docs/design/m3/m3-implementation-plan.md` and
reuses the publication boundaries introduced by
`docs/design/m2/features/simple-write-unit-and-persistence.md`.

This feature is the boundary between the M2 internal write unit and the full M3
transaction model. It must make write ownership, base metadata capture, and
writer cleanup explicit without changing the M0 commit shape.

## 1. Scope

This feature covers:

- one in-process writer gate owned by `DB`
- `beginWrite()` lifecycle checks and fast-fail writer admission
- base metadata capture for the active writer
- stable `base_selected_slot`, `base_generation`, and `base_txn_id` storage on
  `WriteTx`
- explicit `read_only` rejection before any writer state is allocated
- a no-silent-rebase rule between `beginWrite()` and `commit()`
- cleanup rules for rollback, commit success, and terminal commit failure
- tests for exclusivity, lifecycle cleanup, and M4-visible commit assertions
- handoff seams to later M3 dirty-page ownership work and M4 crash validation

This feature does not implement:

- the full dirty-page mutation pipeline
- reader registry cutoff or freelist promotion logic
- crash recovery selection across metadata slots
- cross-process writer locking
- bucket, cursor, or range APIs

## 2. Inputs And Outputs

Inputs:

- a database handle that has completed normal open
- the current in-memory selected metadata snapshot
- the current selected metadata slot
- the database open mode, including whether the handle is read-only
- the transaction lifecycle state for active readers, active writer, and close

Outputs on success:

- exactly one active `WriteTx`
- the writer gate held by that transaction for its full lifetime
- a captured base metadata snapshot stored on the transaction
- the selected slot at begin time copied into `base_selected_slot`
- empty private write state ready for later dirty-page ownership work

Outputs on failure:

- no writer gate leak
- no partial write transaction registered as active
- no mutation-visible state change
- error mapped to `ReadOnlyTransaction`, `WriteConflict`, `InvalidState`, or
  the closest existing lifecycle error

## 3. State Model

The database must make writer ownership observable.

| State | Meaning | Allowed operations |
| --- | --- | --- |
| `writer_idle` | no active write transaction | `beginWrite()` may try to acquire the gate |
| `writer_active` | one `WriteTx` owns the gate and base snapshot | only that transaction may mutate or commit |
| `writer_releasing` | internal cleanup is running after commit or rollback | no new writer may start until cleanup completes |

The transaction stores its own lifecycle state separately:

| `WriteTx` state | Meaning |
| --- | --- |
| `active_write` | gate acquired and base snapshot pinned |
| `committing` | commit started and rollback is no longer allowed |
| `closed_commit` | commit succeeded and cleanup finished |
| `closed_rollback` | rollback released all private state |
| `closed_failed` | commit failed and transaction is terminal |

Implementation rules:

- `DB` owns the writer gate; `WriteTx` only holds the ownership token.
- `beginWrite()` must not return a partially initialized transaction.
- `close()` must reject while the writer gate is held.
- any terminal transaction state must reject later write, read, commit, and
  rollback-dependent mutation entry points with `TransactionClosed`.

## 4. Writer Gate

`beginWrite()` must enforce one active writer without blocking indefinitely.

Required flow:

1. reject if the handle is closing or closed
2. reject if the handle is read-only
3. atomically check and acquire the writer gate
4. capture the current selected metadata snapshot
5. initialize `WriteTx` with empty private state
6. register the transaction as the active writer
7. return the transaction

Writer gate rules:

- a second writer must fail fast with `WriteConflict`
- readers may coexist with the active writer
- the writer gate remains held for the entire write transaction lifetime, not
  only for `commit()`
- helper code below the transaction layer must require a `WriteTx` owner and
  must not allocate dirty state without it
- if initialization fails after the gate is acquired, cleanup must release the
  gate before the error is returned

This feature should prefer a simple atomic flag or small mutex-protected owner
field over a queued lock. M3 needs exclusive ownership, not fairness policy.

## 5. Base Metadata Capture

The active writer must begin from one exact metadata snapshot.

Each `WriteTx` should capture:

- `base_metadata`
- `base_selected_slot`
- `base_generation`
- `base_txn_id`
- `base_root_page_id`
- `base_freelist_page_id`
- `base_page_count`

Capture rules:

- capture happens once, inside `beginWrite()`, after the gate is acquired
- these fields are immutable for the life of the transaction
- later read helpers used by the writer must resolve through the captured base
  snapshot, not through `DB.current_metadata`
- the future commit path must write the inactive slot opposite
  `base_selected_slot`
- the writer must not reread the file or selected slot during commit to choose a
  new base

This is the write-side analogue of the read snapshot rule. The transaction must
plan all copy-on-write work against the same base metadata it observed at begin
time.

## 6. No Silent Rebase

M3 must treat writer base mismatch as a bug or invalid lifecycle state, not as a
reason to adapt the transaction silently.

Required commit assertion before inactive metadata is built:

- `DB.current_metadata.generation == tx.base_generation`
- `DB.current_metadata.txn_id == tx.base_txn_id`
- `DB.selected_slot == tx.base_selected_slot`

Expected behavior on mismatch:

- fail commit with `InvalidState` or a stricter internal assertion path
- keep the in-memory selected metadata unchanged
- transition the transaction to `closed_failed`
- release the writer gate after cleanup

Forbidden behavior:

- rebasing the writer onto newer metadata
- rereading metadata from disk and switching the transaction base
- changing the target inactive metadata slot based on newer in-memory state

This rule is required so M4 can trust the commit path and metadata A/B
alternation observed by M3 tests.

## 7. Read-Only Rejection

The write path must reject read-only handles before any mutation setup begins.

Rules:

- `beginWrite()` on a read-only database returns `ReadOnlyTransaction`
- the function must fail before the writer gate is acquired if the open mode is
  already known
- no private dirty-page map, allocation cursor, or commit hook state may be
  created on this path
- direct M2 compatibility helpers that still expose `DB.put` or `DB.delete`
  must inherit the same rejection behavior through `beginWrite()`

This keeps the write-admission policy centralized and prevents M2 wrapper code
from bypassing the transaction lifecycle.

## 8. Lifecycle Cleanup

Writer cleanup must be complete on every terminal path.

### 8.1 Rollback

Rollback must:

1. reject any new mutation entry point
2. drop private write state
3. clear the active writer registration
4. release the writer gate
5. mark the transaction `closed_rollback`

Rollback invariants:

- no metadata switch
- no disk write, metadata write, sync, or truncation
- no retained writer-owned buffers reachable from shared caches

### 8.2 Commit Success

Commit success must:

1. finish publication through the existing M2-shaped commit flow
2. switch in-memory metadata only after the metadata boundary succeeds
3. clear private write state
4. clear the active writer registration
5. release the writer gate
6. mark the transaction `closed_commit`

### 8.3 Commit Failure

Commit failure must:

1. mark the transaction terminal
2. keep or restore in-memory metadata to the pre-commit selection unless commit
   already succeeded
3. clear private write state that must not survive the failed transaction
4. clear the active writer registration
5. release the writer gate
6. mark the transaction `closed_failed`

Cleanup rules common to all terminal paths:

- the writer gate is released exactly once
- `DB` must not keep a dangling pointer or token to the old transaction
- later `beginWrite()` calls may proceed once cleanup completes, except where a
  higher-level ambiguous-state policy blocks writes
- terminal transactions are never reusable

## 9. Implementation Order

Implement this feature in the following order:

1. add `DB` writer ownership fields and lifecycle assertions
2. add `WriteTx` base snapshot fields and immutable initialization
3. implement `beginWrite()` admission checks for close state, read-only, and
   active writer conflict
4. register and release the writer gate through a small helper API so cleanup is
   testable
5. route any retained M2 write helpers through `beginWrite()` and rollback or
   commit cleanup
6. add the no-silent-rebase commit assertion before metadata publication
7. add rollback, commit-success, and commit-failure cleanup tests
8. document the seam where later M3 work attaches dirty-page ownership and
   freelist-aware commit logic

## 10. Tests

Automated tests should prove lifecycle and ownership behavior rather than only
structure.

| Area | Scenario | Expected result |
| --- | --- | --- |
| Writer gate | first writer begins on writable handle | transaction becomes active and owns the gate |
| Writer gate | second writer begins while first is active | `WriteConflict` |
| Writer gate | writer begins while readers are active | succeeds |
| Read-only | `beginWrite()` on read-only handle | `ReadOnlyTransaction` before writer registration |
| Close lifecycle | `beginWrite()` while DB is closing or closed | `InvalidState` |
| Base capture | writer begins and selected metadata later remains unchanged | `base_*` fields match begin-time metadata |
| Base capture | commit chooses inactive slot | slot is opposite `base_selected_slot` |
| No silent rebase | mutate `DB.current_metadata` in test before commit assertion | commit fails and does not rebase |
| Rollback cleanup | rollback active writer | gate released and later writer may begin |
| Commit cleanup | successful commit | gate released, writer cleared, tx closed |
| Failed cleanup | injected pre-publication or metadata-path failure | gate released, tx closed failed |
| Wrapper behavior | retained `DB.put` or `DB.delete` helper | internally acquires and releases one writer transaction |
| Reuse rejection | use writer after commit, rollback, or failure | `TransactionClosed` |
| Close rejection | call `close()` with active writer | `InvalidState` |

Recommended executable test groups:

- `single_writer_gate`
- `write_base_snapshot`
- `write_lifecycle_cleanup`
- `write_no_silent_rebase`
- `write_read_only_rejection`

## 11. M4 Handoff

This feature must leave these seams intact for M4:

- the commit path still publishes through one inactive metadata write
- the target metadata slot is derived from `base_selected_slot`, not from a late
  re-read
- failure before in-memory metadata publication leaves the old in-memory
  selection authoritative for the current process
- terminal cleanup never leaves the writer gate held after an injected failure
- failpoints can distinguish pre-metadata failure from ambiguous metadata-path
  failure without changing writer ownership rules

M4 should be able to add stricter durability and reopen validation without
replacing `beginWrite()`, writer exclusivity, or the no-silent-rebase rule.

## 12. Acceptance

This feature is complete when:

- `beginWrite()` allows at most one active writer in the process
- a second writer fails with `WriteConflict` and does not block indefinitely
- `beginWrite()` on a read-only handle fails with `ReadOnlyTransaction`
- every active writer stores one immutable base metadata snapshot
- commit targets the inactive slot opposite `base_selected_slot`
- commit detects base metadata mismatch and fails without silent rebase
- rollback, commit success, and commit failure all release the writer gate
- terminal cleanup clears active writer registration and private state
- `close()` rejects while a writer is active
- retained direct M2 write helpers, if any remain public, route through the same
  writer gate and rejection rules
- tests cover exclusivity, base capture, no-silent-rebase, read-only rejection,
  lifecycle cleanup, and terminal transaction reuse rejection
