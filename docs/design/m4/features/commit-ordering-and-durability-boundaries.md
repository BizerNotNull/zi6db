# M4 Feature Plan: Commit Ordering And Durability Boundaries

## Goal

Define the concrete M4 implementation plan for the commit path rules frozen by M0:

- all non-metadata writes happen before metadata publication
- pre-metadata and post-metadata sync boundaries are explicit
- metadata remains the only durable publication switch
- `strict`, `normal`, and `unsafe` preserve the same logical ordering
- commit failures map cleanly to Class A, B, and C outcomes

This feature does not change the on-disk format. It turns the M0 ordering contract into a precise M4 implementation and test target.

## Frozen Inputs

This plan must follow:

- `docs/design/m0/page-lifecycle-and-commit-ordering.md`
- `docs/design/m4/m4-implementation-plan.md`
- the repository rules in `AGENTS.md`

The implementation must preserve these frozen M0 facts:

- visible pages are never overwritten in place
- metadata is the only commit switch point
- `generation` is the primary metadata ordering field
- any failure before in-memory metadata publication leaves the current process on the old metadata snapshot

## Scope

This feature covers:

- ordering for dirty branch, leaf, overflow, and freelist page writes
- the pre-metadata sync boundary
- inactive metadata construction and write ordering
- the post-metadata sync boundary
- the in-memory publication boundary
- durability mode behavior for `strict`, `normal`, and `unsafe`
- failure classification into Class A, Class B, and Class C
- crash, corruption, and sync-policy tests

This feature does not cover:

- metadata selection rules beyond what commit ordering must preserve for recovery
- new on-disk structures
- repair or salvage of corrupted files
- user-facing tooling

## Required Invariants

The M4 implementation must keep these invariants explicit in code and tests:

1. No non-metadata write may target a page reachable from the writer base metadata snapshot.
2. All referenced branch, leaf, overflow, and freelist pages must be finalized before metadata is built.
3. The inactive metadata page must be written only after every referenced non-metadata page write has completed.
4. The pre-metadata sync boundary must complete before the metadata write in any mode that promises crash-recoverable success.
5. The post-metadata sync boundary must complete before in-memory publication in any mode that promises durable committed state.
6. In-memory publication is a separate step after metadata persistence work, not part of metadata encoding.
7. No fallible persistence step may remain after in-memory publication.
8. Commit success means both the selected mode's durability promise and in-memory publication have completed.

## Implementation Shape

Implement the commit path as a small state machine with stable phase names:

- `reserve_tail_pages`
- `write_data_pages`
- `sync_data_pages`
- `build_metadata`
- `write_metadata`
- `sync_metadata`
- `publish_metadata`

The state machine should be shared by normal commit logic and fault-injection tests so we do not maintain two different definitions of the boundary behavior.

## Non-Metadata Writes

### Write Set

The non-metadata write set for one commit includes:

- cloned or newly allocated branch pages
- cloned or newly allocated leaf pages
- overflow pages, when present
- the new freelist page

### Ordering Rules

The implementation order should be:

1. reserve `new_txn_id` and `new_generation` from the writer base metadata
2. finalize dirty tree pages
3. finalize freelist contents, including `reusable` and `pending_free`
4. compute page checksums after page contents are final
5. write every non-metadata page with positional whole-page writes

The write path must treat file growth and page writes as one boundary class. Tail extension failure, short page write, or torn normal-page write before metadata begins is still a pre-publication failure.

### Safety Rules

- A commit must not reuse pages freed by the same transaction.
- A commit must not write a page id that is still reachable from the base metadata snapshot.
- Unreachable tail pages created by a failed commit are allowed, but they must stay unreachable unless a later valid metadata slot references them.

## Pre-Metadata Sync Boundary

The pre-metadata sync boundary exists to prevent recovery from selecting metadata that references pages not durably written first within the promised failure model.

### Required Behavior

- `strict` must execute the full pre-metadata sync.
- `normal` must initially alias `strict`.
- `unsafe` may skip the durable flush, but it may not reorder metadata before the non-metadata write phase.

### Failure Meaning

If pre-metadata sync fails:

- `commit()` returns `IoError`
- the transaction becomes terminal
- the current process keeps the old metadata snapshot
- reopen must select the old metadata, because the metadata write never started

This is a Class A failure.

## Metadata Construction And Write

### Build Rules

Build the inactive metadata page only after all referenced values are known:

- `root_page_id`
- `freelist_page_id`
- `page_count`
- `new_generation`
- `new_txn_id`
- inactive `slot_id`

The metadata checksum must be computed after every other metadata field is final.

### Publication Boundary

The metadata page write is the only durable publication boundary on disk. Once this phase starts, the commit may become externally ambiguous until metadata sync is confirmed.

### Write Rules

- always write the inactive slot determined from the writer base selected slot
- write the metadata page with one positional whole-page write
- do not switch in-memory metadata here

Any failure after metadata write starts but before metadata sync is confirmed is Class B.

## Post-Metadata Sync Boundary

The post-metadata sync boundary confirms whether the inactive metadata write crossed the mode's persistence guarantee.

### Required Behavior

- `strict` must not return success before metadata sync completes
- `normal` must behave the same as `strict` until a lighter path is separately justified and documented
- `unsafe` may skip the durable flush, but it must still preserve the phase order `write_data_pages -> build_metadata -> write_metadata -> publish_metadata`

### Failure Meaning

If metadata sync fails or commit crashes after metadata write but before metadata sync confirmation:

- `commit()` returns `IoError` when the process observes the failure
- the transaction becomes terminal
- the current process keeps the old metadata snapshot
- reopen may select either old or new metadata, but only through normal validation and generation comparison

This is Class B.

## In-Memory Publication Boundary

In-memory publication is the final step:

1. all required write phases completed
2. all required sync phases for the selected mode completed
3. current metadata pointer switches to the new snapshot
4. transaction closes and writer lock is released

This boundary exists even in `unsafe`. `unsafe` may weaken power-loss durability, but it may not expose partial state to concurrent readers before `commit()` returns success.

After publication, no additional fallible I/O for the same commit may run. A commit that reaches publication and returns success is Class C.

## Durability Mode Semantics

| Mode | Non-metadata writes | Pre-metadata sync | Metadata write | Post-metadata sync | Success promise |
| --- | --- | --- | --- | --- | --- |
| `strict` | required | required | required | required | committed state survives within the platform failure model |
| `normal` | required | alias `strict` initially | required | alias `strict` initially | same as `strict` until documented otherwise |
| `unsafe` | required | optional | required | optional | no power-loss durability promise, but no in-process partial visibility |

### Mode Rules

- `strict` is the reference implementation for M4.
- `normal` must not diverge until the lighter path is backed by platform evidence and tests.
- `unsafe` weakens only durable flush requirements. It does not weaken copy-on-write, checksum finalization, metadata slot alternation, or publication ordering.

## Class A/B/C Mapping

| Class | Boundary | Typical examples | API result | Current-process metadata | Reopen result |
| --- | --- | --- | --- | --- | --- |
| A | before metadata write begins | file growth failure, normal-page short write, pre-metadata sync error | `IoError` | stays old | old metadata must win |
| B | metadata write may have started, metadata sync not confirmed | torn metadata write, crash after metadata write, sync uncertainty | `IoError` if observed | stays old | old or new may win by validation |
| C | metadata sync confirmed and publication completed | successful commit | success | switches to new | new metadata must win within the mode promise |

### Required Interpretation

- Class A is unambiguous non-commit.
- Class B is intentionally ambiguous on disk.
- Class C is the only successful commit outcome.

Tests and comments should describe Class B carefully: an `IoError` after metadata write starts is not proof that the new commit is absent on disk.

## Test Plan

### Non-Metadata Write Tests

- fail before any data-page write and verify old generation remains selected
- fail during a branch, leaf, overflow, or freelist page write and verify metadata is unchanged
- crash after all non-metadata writes but before pre-metadata sync and verify old metadata still wins
- create unreachable tail pages through injected failure and verify reopen ignores them

### Sync Boundary Tests

- verify `strict` calls both sync boundaries before success
- verify pre-metadata sync failure stays Class A
- verify metadata sync failure stays Class B
- verify `unsafe` skips allowed flushes without reordering the logical phases
- verify `normal` currently matches `strict`

### Publication Boundary Tests

- verify current-process metadata does not switch before `publish_metadata`
- verify a Class B failure leaves current-process readers on the old snapshot
- verify success publishes exactly once and only after required sync completion
- verify no fault injection point exists after `publish_metadata` for the same commit I/O path

### Class Mapping Tests

- one test per Class A trigger family
- one test per Class B trigger family
- one success-path Class C test per durability mode
- each test must assert both immediate commit result and reopened database result

### Recovery-Facing Tests

- torn or checksum-invalid metadata written during Class B is rejected on reopen
- valid newer metadata written before crash may win on reopen in Class B
- old valid metadata wins when the newer slot references a bad root or freelist page

## Acceptance Criteria

- non-metadata pages are always written before metadata for every mode
- `strict` and initial `normal` perform the pre-metadata and post-metadata sync boundaries
- `unsafe` preserves logical write ordering even when it skips flushes
- metadata is built only after final root, freelist, page count, generation, and transaction id are known
- metadata checksum is finalized last
- in-memory metadata publication happens only after the mode's required persistence work completes
- commit failure before publication always leaves the current process on the old metadata snapshot
- Class A failures reopen to the old metadata only
- Class B failures keep the old in-memory snapshot but allow reopen to choose old or new through validation
- Class C success returns only after the new metadata is both durable enough for the selected mode and published in memory
- tests cover non-metadata writes, both sync boundaries, publication ordering, durability modes, and Class A/B/C outcomes

## Delivery Notes

When implementation starts, keep the commit phase names stable across code, tests, and future checker output. That stability matters because this feature is fundamentally about proving boundaries, not just performing I/O in the right order once.
