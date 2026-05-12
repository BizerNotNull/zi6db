# M2 Feature Plan: Simple Write Unit And Persistence

This plan defines the concrete M2 implementation work for persisting one
successful `put` or one successful `delete` as an internal single-operation
write unit. It refines the M2 implementation plan while preserving the frozen
M0 metadata-switching and recovery rules.

M2 does not expose public transactions. The simple write unit is an internal
commit-shaped boundary that makes each successful mutation visible only through
the inactive metadata slot.

## 1. Scope

This feature covers:

- tail allocation for cloned B+Tree pages, split pages, new roots, and the new
  freelist page
- a freelist placeholder that can preserve or introduce pending-free state
  without implementing page reuse
- ordered non-metadata page writes for staged branch, leaf, and freelist pages
- durability-mode sync hooks before and after metadata publication
- writing the inactive metadata slot as the only commit switch point
- in-memory metadata publication only after the metadata write and required sync
  report success
- conservative failure handling, including an ambiguous failure state that
  requires reopen before later writes
- persistence tests for clean close and reopen
- handoff seams for M3 transactions and M4 crash/fault injection

This feature does not implement:

- public transaction objects
- rollback of already-written disk bytes
- freelist reuse
- power-loss fault-injection guarantees
- page repair
- multi-operation atomicity

## 2. Inputs And Outputs

Inputs:

- a writable database handle
- the selected in-memory metadata snapshot and selected slot from `open()`
- a staged B+Tree mutation result from the M2 mutation planner
- the current durability mode

Outputs on success:

- all newly reachable branch, leaf, root, and freelist pages written to disk
- inactive metadata slot written with `generation + 1` and `txn_id + 1`
- in-memory selected metadata switched to the new metadata snapshot
- selected slot switched to the slot that was just written
- old copied pages recorded as pending-free if the freelist codec supports it

Outputs on failure:

- no in-memory root, freelist, `generation`, `txn_id`, or selected slot switch
- writer guard released
- dirty staged pages discarded from process state
- public error mapped to `IoError`, `ReadOnlyTransaction`, `InvalidState`, or the
  closest existing M1/M2 error
- if metadata write may have started without confirmed metadata sync, the handle
  enters `reopen_required` before any later write may begin

## 3. State Model

The database handle should distinguish three write-unit states:

| State | Meaning | Allowed operations |
| --- | --- | --- |
| `ready` | no active writer and selected metadata is authoritative in memory | reads and writes according to handle mode |
| `writing` | one internal write unit owns the writer guard and private staged pages | only the active write unit may proceed |
| `reopen_required` | a metadata write or metadata sync failed ambiguously | reads follow the documented policy; writes fail until close and reopen |

`reopen_required` is not corruption by itself. It means the process cannot know
whether recovery will select old metadata or new metadata without running the
normal M0/M1 open path again.

## 4. Tail Allocation

M2 allocation is tail-only.

Implementation rules:

1. Capture `base_page_count` from the selected metadata before mutation
   publication begins.
2. Allocate each new physical page id from a monotonic local cursor initialized
   to `base_page_count`.
3. Allocate all cloned root-to-leaf pages from the tail, even when the logical
   page being cloned is later pending-free.
4. Allocate split siblings and any new branch root from the same tail cursor.
5. Allocate the new freelist page from the tail after tree pages are known, so
   metadata can reference one concrete freelist page id.
6. Set `new_page_count` to the final tail cursor.
7. Do not reduce `page_count`, even after deletes.
8. Do not infer reusable pages from file length, complete tail pages, partial
   tail residue, or unreachable pages.

Tail allocation may leave orphaned pages after failed writes. Those pages are
unreachable from selected metadata and must be ignored by `open()`.

## 5. Freelist Placeholder And Pending-Free

M2 is not a page-reuse milestone, but every successful write must keep metadata
and freelist ownership coherent.

The write unit should build a new freelist page for every successful mutating
write, even if the freelist body is still minimal. This keeps the copy-on-write
rule simple: metadata always points to a newly written freelist page that belongs
to the new generation.

Freelist behavior:

- Preserve any freelist state that the selected freelist page can already decode.
- Add copied old root-path pages and the old selected freelist page to
  `pending_free[new_txn_id]` if pending groups are supported by the codec.
- If M1 only has an empty freelist placeholder, extend it with the smallest
  pending-free group encoding needed for M2 tests; do not add reusable-page
  allocation.
- Do not put pages freed by logical delete into `reusable` in the same write.
- Do not allocate from `pending_free` or `reusable`.
- Do not require file size shrinkage in any test or acceptance rule.

Pending-free records are a handoff artifact for M3/M6. Lookup correctness in M2
must not depend on reading pending-free state.

## 6. Data Writes

The mutation planner should produce encoded full-page images plus enough metadata
to publish them:

- `new_root_page_id`
- dirty branch and leaf page images
- dirty freelist page image
- old copied page ids for pending-free accounting
- `new_page_count`

The simple write unit writes pages in this order:

1. Finalize checksums for all dirty branch and leaf pages.
2. Finalize the checksum for the new freelist page.
3. Write all dirty branch and leaf pages with positional whole-page writes.
4. Write the new freelist page with a positional whole-page write.
5. Treat short writes as `IoError`.
6. Do not write metadata until every non-metadata page write has succeeded.

The order among non-metadata tree pages does not define visibility because none
are reachable until metadata switches. The freelist page must still be written
before metadata because the new metadata references it.

## 7. Sync Hooks

The write unit must call the M1 storage sync wrappers rather than embedding
platform-specific sync logic.

Required hook points:

1. `sync_data_before_metadata(durability_mode)` after all non-metadata pages are
   written and before inactive metadata is written.
2. `sync_metadata_after_write(durability_mode)` after the inactive metadata page
   is written.

Durability modes may map these hooks differently, but the call sites must remain
stable so M4 can inject faults at the same boundaries.

If a platform cannot provide the strict M0 durability behavior yet, the
implementation notes and tests must name that limitation. The write unit still
must preserve the same logical ordering.

## 8. Inactive Metadata Publication

Metadata is the only commit switch point.

Publication rules:

1. Determine the inactive slot from the selected in-memory slot.
2. Build a metadata page for that inactive slot only.
3. Set `generation = base.generation + 1`.
4. Set `txn_id = base.txn_id + 1`.
5. Set `root_page_id = mutation.new_root_page_id`.
6. Set `freelist_page_id = new_freelist_page_id`.
7. Set `page_count = mutation.new_page_count`.
8. Preserve immutable fields duplicated from the validated header.
9. Finalize the metadata checksum after all fields are encoded.
10. Write exactly one inactive metadata page for the logical mutation.
11. Never rewrite the currently selected active metadata slot during this write.
12. Never create equal-generation metadata slots.

Only after the inactive metadata write and metadata sync hook report success may
the handle switch its in-memory selected metadata and selected slot.

## 9. Failure Semantics

Failures are classified by how far publication progressed.

### 9.1 Before Any Disk Write

Examples:

- read-only handle
- entry too large
- allocation overflow
- mutation planning error
- checksum encoding failure before I/O

Expected behavior:

- return the mapped error
- release the writer guard
- discard staged pages
- keep in-memory selected metadata unchanged
- leave the handle in `ready`

### 9.2 Non-Metadata Write Or Pre-Metadata Sync Failure

Examples:

- branch or leaf page write failure
- freelist page write failure
- short write
- `sync_data_before_metadata` failure

Expected behavior:

- return `IoError`
- release the writer guard
- discard staged pages
- keep in-memory selected metadata unchanged
- keep the handle in `ready`

Recovery must select the old metadata because no metadata write began. Any
orphaned tail pages are unreachable residue.

### 9.3 Ambiguous Metadata Write Or Metadata Sync Failure

Examples:

- inactive metadata write reports a partial or uncertain failure
- inactive metadata write returns an I/O error after issuing the write
- `sync_metadata_after_write` fails

Expected behavior:

- return `IoError`
- release the writer guard
- discard staged pages
- keep in-memory selected metadata unchanged
- set handle state to `reopen_required`
- reject later `put` and successful `delete` attempts with `InvalidState` until
  the database is closed and reopened

The on-disk outcome is ambiguous. Reopen may select either the old metadata or
the new metadata according to the normal M0/M1 metadata selection rules.

### 9.4 Reads In Reopen-Required State

M2 should choose one explicit policy and test it:

- Preferred policy: allow `get`, `exists`, and `check` to read only from the old
  in-memory selected metadata snapshot while writes fail with `InvalidState`.
- Acceptable stricter policy: fail all public operations except `close()` with
  `InvalidState`.

The preferred policy is useful for callers, but it must not inspect dirty staged
pages or the possibly-written inactive metadata. Reopen remains the only
authoritative persisted-state decision.

## 10. Write Flow

The complete implementation flow for `put` and successful `delete` is:

1. Reject if the handle is read-only.
2. Reject if handle state is `reopen_required`.
3. Acquire the writer guard.
4. Set handle state to `writing`.
5. Capture base metadata and selected slot.
6. Plan the B+Tree mutation using copy-on-write pages.
7. If `delete` finds no key, discard the plan, restore `ready`, release the
   guard, and return `false` without writing pages or metadata.
8. Allocate tail page ids for all staged tree pages.
9. Build the new freelist placeholder or pending-free page.
10. Encode and checksum all non-metadata pages.
11. Write all non-metadata pages.
12. Call the pre-metadata sync hook.
13. Encode and checksum inactive metadata.
14. Write inactive metadata.
15. Call the metadata sync hook.
16. Switch in-memory selected metadata and selected slot.
17. Restore handle state to `ready`.
18. Release the writer guard.

No step before step 16 may mutate the selected in-memory metadata snapshot.

## 11. Tests

Automated tests should use names that describe behavior, not implementation
details.

| Area | Required cases |
| --- | --- |
| Tail allocation | first write allocates pages at old `page_count`; multiple writes monotonically increase `page_count`; deletes do not shrink the file |
| Data persistence | `put`, replace, delete, leaf split, root split, and mixed workloads survive close and reopen |
| Metadata switch | success writes the inactive slot; selected slot alternates; `generation` and `txn_id` increment together |
| Missing delete | deleting an absent key returns `false` and does not publish a new metadata generation |
| Freelist placeholder | new metadata references a valid freelist page; old copied pages are pending-free when the codec supports pending groups |
| Non-metadata failure | injected tree or freelist write failure leaves old in-memory metadata selected and allows a later write |
| Pre-metadata sync failure | sync failure before metadata leaves old in-memory metadata selected and allows a later write |
| Metadata write failure | ambiguous inactive metadata failure keeps old in-memory metadata selected and enters `reopen_required` |
| Metadata sync failure | failed post-metadata sync keeps old in-memory metadata selected and enters `reopen_required` |
| Reopen required | later writes fail before mutation planning; documented read policy is enforced |
| Reopen authority | after an ambiguous failure, closing and reopening selects the newest valid metadata or the old metadata according to M0 rules |
| Tail residue | orphaned complete tail pages and partial tail bytes after failed writes are ignored by open and not inserted into the freelist |
| Read-only | `put` and `delete` fail before allocation or page writes |
| Checker integration | checker passes after successful persisted writes and after reopen |

Fault injection can be implemented with a mock storage layer or deterministic
write/sync failpoints. M2 tests do not need to simulate power loss; they only
need to prove the in-process state machine and reopen behavior.

## 12. Acceptance

This feature is complete when:

- every successful `put` and successful `delete` publishes through exactly one
  inactive metadata write
- no successful write overwrites the visible root, freelist, branch, or leaf
  pages in place
- allocation is tail-only and does not reuse freelist pages
- metadata `generation` and `txn_id` increase together on successful writes
- missing-key `delete` does not allocate, write, sync, or increment metadata
- dirty staged pages are never visible to reads before metadata publication
- failed writes before metadata do not switch memory and do not poison later
  writes
- ambiguous metadata write or sync failure leaves memory on the old metadata and
  blocks later writes until reopen
- clean close and reopen preserve inserted, replaced, and deleted key state
- open remains authoritative for ambiguous persisted outcomes
- test coverage includes write ordering, sync failpoints, inactive metadata,
  freelist placeholder behavior, and reopen-required state

## 13. M3 And M4 Handoff

M3 should be able to wrap this simple write unit as the commit path for a real
write transaction. To keep that possible:

- keep mutation planning separate from publication
- keep staged dirty pages private until the publication function runs
- represent old copied pages explicitly for future rollback and pending-free
  cutoff logic
- keep the in-memory metadata switch centralized in one function
- preserve the `reopen_required` terminal state for ambiguous commit failures

M4 should be able to add crash and corruption testing without changing the write
unit boundaries. To keep that possible:

- keep failpoint seams around non-metadata writes, pre-metadata sync, metadata
  write, and metadata sync
- keep inactive metadata as the only on-disk visibility switch
- keep orphaned tail pages unreachable from selected metadata
- keep startup recovery delegated to the M0/M1 open path
- do not claim power-loss safety beyond the durability mode actually implemented

M6 can later replace tail-only allocation with safe freelist reuse. Until then,
the freelist data written by M2 is a correctness-preserving placeholder and a
future ownership record, not a source of reusable page ids.
