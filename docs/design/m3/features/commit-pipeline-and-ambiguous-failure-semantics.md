# M3 Feature Plan: Commit Pipeline And Ambiguous Failure Semantics

This plan defines the concrete M3 implementation work for the write-transaction
commit path and its failure semantics. It refines
`docs/design/m3/m3-implementation-plan.md` using the recovery and durability
boundaries reserved for M4 in `docs/design/m4/m4-implementation-plan.md`.

The goal is to make M3 commits correct in-process now while preserving the exact
shape that M4 will harden for crash safety and deterministic reopen behavior.

## 1. Scope

This feature covers:

- the ordered commit phases for one `WriteTx`
- the inactive metadata slot rule and strict A/B alternation
- the publication boundary between staged state and visible committed state
- failure classification into Class A, Class B, and Class C
- durability hook boundaries that M4 will later strengthen
- tests for ordering, visibility, and ambiguous-failure behavior
- explicit handoff requirements for M4 recovery and fault injection

This feature does not cover:

- startup crash recovery selection logic
- new on-disk format rules
- cross-process locking
- background cleanup or repair
- freelist reuse policy beyond what M3 already needs for reader safety

## 2. Inputs And Frozen Constraints

This feature must preserve these already-frozen rules:

- metadata is the only durable commit switch point
- a write transaction starts from one stable base metadata snapshot
- the writer must not silently rebase during commit
- visible pages are never overwritten in place
- the next commit writes the inactive metadata slot opposite the base slot
- `generation` and `txn_id` both increment exactly once on successful commit
- in-memory publication happens only after the metadata boundary succeeds
- after an ambiguous failure, reopen is the only authority for persisted state

## 3. Commit Outcome Model

M3 should make one distinction explicit in code and tests:

- staged state: dirty pages, pending freelist changes, and the candidate
  metadata page exist only inside the active `WriteTx`
- published state: the `DB` current metadata snapshot points at the new root,
  new freelist, new `generation`, and new `txn_id`

The publication boundary is the transition from staged state to published state.
Nothing before that boundary may make the new commit visible to readers in the
current process.

## 4. Commit Phases

Implement the commit path as a small ordered phase machine. Stable phase names
keep tests readable and give M4 exact failpoint seams.

| Phase | Purpose | Required invariant |
| --- | --- | --- |
| `prepare_commit` | freeze base metadata, reserve `new_generation` and `new_txn_id`, choose inactive slot | base metadata and base slot still match `DB` current metadata |
| `promote_freelist_state` | move only reader-safe pending-free groups into reusable and apply this transaction's frees under `pending_free[new_txn_id]` | pages freed by this transaction are not reusable in the same commit |
| `finalize_data_pages` | finalize branch, leaf, overflow-if-any, and freelist page images including generation/header/checksum fields | all non-metadata page bytes are final before any write begins |
| `write_data_pages` | write all non-metadata pages using copy-on-write page ids | no write targets a page reachable from base metadata |
| `sync_data_pages` | run the pre-metadata durability hook for the selected durability mode | metadata is still unwritten |
| `build_metadata` | encode inactive-slot metadata with new root, freelist, page count, generation, and txn id | metadata checksum is computed last |
| `write_metadata` | write exactly one inactive metadata page | active slot is not rewritten |
| `sync_metadata` | run the metadata durability hook for the selected durability mode | on-disk result may still be ambiguous until this succeeds |
| `publish_metadata` | switch in-memory current metadata and selected slot | this is the first visibility point for new readers in the current process |
| `finish_commit` | clear private writer state, release the writer gate, mark closed-commit | no later fallible persistence step remains |

## 5. Inactive Slot Rule

The commit path must determine the inactive slot from the write transaction's
captured base slot, not by re-reading the file during commit.

Rules:

- base slot A commits to slot B
- base slot B commits to slot A
- the commit path writes exactly one metadata slot
- the currently selected active slot is never rewritten as part of the same
  commit
- equal-generation dual-slot states remain impossible in normal operation

This rule is a correctness boundary, not a convenience detail. M4 recovery
depends on M3 already preserving strict A/B alternation.

## 6. Publication Boundary

The publication boundary is after `sync_metadata` succeeds and before any new
transaction can capture the new metadata snapshot.

Before the publication boundary:

- existing readers keep their pinned snapshots
- new readers still begin from the old metadata snapshot
- the current process must treat the new commit as not visible
- any `commit()` failure returns with old in-memory metadata still selected

After the publication boundary:

- new readers begin from the new metadata snapshot
- the writer transaction is considered successfully committed
- the API must not report commit failure for the same write

Implementation rule:

- `publish_metadata` must be a separate step, not an accidental side effect of
  `write_metadata` or `sync_metadata`

## 7. Durability Hooks

M3 must keep the hook boundaries explicit even if the first implementation maps
them to the existing M1/M2 sync behavior.

Required hooks:

1. `sync_data_before_metadata(durability_mode)`
2. `sync_metadata_after_write(durability_mode)`

Hook rules:

- the data hook runs only after all non-metadata pages are fully written
- the metadata hook runs only after the inactive metadata page is fully written
- hook failures propagate as commit failure and close the transaction into a
  terminal failed state
- M3 must not collapse both hooks into one opaque `sync()` call at the commit
  API boundary, because M4 needs separate Class A and Class B failpoints

## 8. Failure Classes

Commit failures are classified by how far the pipeline progressed.

### 8.1 Class A

Class A means the metadata write did not begin.

Examples:

- page allocation failure during commit preparation
- freelist finalization failure
- non-metadata page write failure
- pre-metadata data sync failure

Expected behavior:

- `commit()` returns the mapped error, typically `IoError`
- in-memory current metadata does not switch
- the write transaction becomes terminal failed
- the writer gate is released after private-state cleanup
- a later reopen is expected to select the old metadata because no metadata
  write began

### 8.2 Class B

Class B means the metadata write may have started, but metadata sync was not
confirmed.

Examples:

- metadata write returns a short or uncertain result
- metadata write reports failure after issuing bytes
- metadata sync returns error after metadata write

Expected behavior:

- `commit()` returns `IoError`
- in-memory current metadata does not switch
- the write transaction becomes terminal failed
- the writer gate is released after cleanup
- the on-disk result is treated as ambiguous
- the current process must not claim to know whether the commit persisted

The key semantic is that reopen becomes authoritative. A later `open()` may
select old or new metadata depending on which slot validates and wins by
generation.

### 8.3 Class C

Class C means metadata sync completed for the selected durability mode.

Expected behavior:

- `commit()` proceeds to in-memory publication
- `commit()` returns success only after publication completes
- no later cleanup issue may be reported as a commit failure that contradicts
  the durable success point

## 9. Transaction State Effects

This feature should keep the state transitions narrow and testable:

| From | Event | To |
| --- | --- | --- |
| `active_write` | `commit()` starts | `committing` |
| `committing` | Class C success and publication complete | `closed_commit` |
| `committing` | Class A failure | `closed_failed` |
| `committing` | Class B failure | `closed_failed` |

State rules:

- once `committing` begins, `rollback()` is no longer a recovery tool
- `closed_failed` is terminal and unreusable
- later operations on a failed writer return `TransactionClosed`

## 10. Implementation Plan

Implement this feature in the following order:

1. Add a commit phase enum with the stable phase names in this document.
2. Record the writer base slot and assert it still matches the selected
   in-memory metadata before metadata build begins.
3. Isolate freelist promotion and `pending_free[new_txn_id]` application into a
   pre-write finalization step.
4. Finalize all normal dirty pages before any disk write starts.
5. Route all non-metadata writes through one ordered `write_data_pages` step.
6. Add the pre-metadata durability hook call.
7. Build inactive-slot metadata only after data-page finalization succeeds.
8. Add the metadata write step and metadata durability hook call.
9. Add a dedicated `publish_metadata` step that switches in-memory metadata and
   selected slot only after the metadata hook succeeds.
10. Ensure all failure exits release the writer gate, clear private dirty state,
    and end in `closed_failed`.

## 11. Tests

Tests should name the phase or semantic being proven.

| Area | Scenario | Expected result |
| --- | --- | --- |
| Commit phases | injected failure in `write_data_pages` | old in-memory metadata remains selected; transaction ends failed |
| Commit phases | injected failure in `sync_data_pages` | old in-memory metadata remains selected; failure is Class A |
| Commit phases | injected failure in `write_metadata` | old in-memory metadata remains selected; failure is Class B |
| Commit phases | injected failure in `sync_metadata` | old in-memory metadata remains selected; failure is Class B |
| Commit phases | success through `publish_metadata` | new in-memory metadata becomes visible only after metadata sync |
| Inactive slot | base slot A commit | metadata writes slot B |
| Inactive slot | base slot B commit | metadata writes slot A |
| Publication boundary | reader begins before commit success | reader keeps old snapshot |
| Publication boundary | reader begins after commit success | reader sees new snapshot |
| Publication boundary | metadata write started but Class B returned | current process still reads old metadata |
| Failure classes | Class A commit error | later reopen is expected to select old metadata |
| Failure classes | Class B commit error | commit returns `IoError`; reopen remains authoritative |
| Durability hooks | strict or current alias mode | data hook fires before metadata hook |
| State model | reuse writer after Class A or B | `TransactionClosed` |

Minimum executable test group names:

- `commit_phase_ordering`
- `inactive_slot_selection`
- `publication_boundary`
- `ambiguous_failure_semantics`
- `durability_hook_boundaries`

## 12. M4 Handoff

M4 should be able to adopt this commit path without replacing it.

To keep that possible, M3 must leave these seams intact:

- stable phase names for fault injection and crash-matrix tests
- explicit pre-metadata and post-metadata durability hooks
- base-slot-driven inactive-slot selection
- a separate in-memory publication step after metadata sync
- Class B behavior where the current process stays on old metadata while reopen
  may choose old or new
- no fallible persistence work after publication

M4 then adds:

- platform-specific strict durability mappings
- fault injection before and after each durable boundary
- reopen-based validation of Class A, B, and C outcomes
- checker and metadata-slot validation for ambiguous outcomes

## 13. Acceptance

This feature is complete when:

- the write commit path is implemented as ordered phases rather than one opaque
  helper
- the inactive metadata slot is chosen from the writer base slot and alternates
  A/B strictly
- non-metadata pages are written before any metadata write begins
- the pre-metadata and post-metadata durability hooks are explicit and testable
- in-memory metadata is published only after metadata sync succeeds
- Class A failures leave memory on the old metadata and are treated as
  unambiguously uncommitted in-process
- Class B failures leave memory on the old metadata and are documented as
  ambiguous on disk
- Class C reaches a durable success point before publication and returns success
  only after publication
- failed writers become terminal and unreusable
- tests cover commit phases, failure classes, publication boundary, durability
  hooks, inactive slot behavior, and M4 handoff seams
