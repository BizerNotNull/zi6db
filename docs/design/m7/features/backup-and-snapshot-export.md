# Backup And Snapshot Export

## Goal

This document defines the concrete M7 implementation plan for the required
backup feature. The M7 backup path must export a consistent read-only snapshot
of the database without changing source state, reassigning page ids, promoting
free pages, or performing compaction.

The required M7 mode is an identity-preserving snapshot copy. Every copied page
keeps its original page id, and the output must remain compatible with normal
`open()` and `verify()` behavior.

## Scope And Invariants

The backup implementation must preserve all M0 and M6 rules that matter to a
visible snapshot:

- Backup starts from a pinned read transaction snapshot.
- Backup copies only the state visible to that snapshot.
- Backup never reads uncommitted writer state as committed output.
- Backup preserves page ids for all copied pages.
- Backup preserves the pinned snapshot's root, freelist, overflow chains,
  `reusable` ranges, and `pending_free` groups exactly as visible in that
  snapshot.
- Backup does not reclaim space, compact pages, infer a new freelist, or repair
  corruption.
- Backup does not include ignored tail pages beyond the pinned `page_count`.
- Backup output must contain exactly one valid metadata slot.

If the engine later grows a rewrite mode that changes page ids, that mode is
compaction, not backup, and must stay outside this document's scope.

## Pinned Snapshot Model

### Snapshot Start

Backup must begin by opening a normal read transaction through the same path
used by user reads. The read transaction pins:

- the selected metadata slot
- the visible `generation`
- the visible `txn_id`
- the visible `page_count`
- the visible root page id
- the visible freelist root page id

This snapshot is authoritative for the entire backup. Later commits from a
concurrent writer must not change any page selection, page content, freelist
state, or metadata fields copied into the backup.

### Snapshot Lifetime

The read transaction stays open until the destination file has either:

- been fully written and synced according to options, or
- failed and completed cleanup

The implementation must release the read transaction on every exit path,
including destination create failure, mid-copy I/O failure, sync failure, and
temporary-file cleanup failure.

### Snapshot Coordination With Writers

Backup must not require writer shutdown. The only allowed coordination is the
same lock and snapshot machinery already used to provide a stable reader view.

Writers may commit after backup starts, but those later commits must not appear
in the output. Writers must also remain free to reuse M0/M6 copy-on-write
behavior without backup chasing newly published metadata.

## Page-ID Preserving Copy Plan

### Reachability Enumeration

After pinning the snapshot, backup must enumerate every page reachable from the
selected metadata state. The traversal must reuse the M6 live-page
classification and validators rather than inventing a second ownership model.

Required reachable categories:

- selected root and all reachable branch and leaf pages
- bucket catalog pages and bucket-owned trees
- overflow chains reachable from visible leaf values
- selected freelist pages
- `reusable` page references recorded by the selected freelist state
- `pending_free` groups recorded by the selected freelist state

The output must therefore preserve the source snapshot's future-safe reuse
state, not just current logical key/value contents.

### Copy Layout

The destination file keeps source page numbers unchanged:

- page `0` remains the header page
- metadata pages remain at their normal metadata page ids
- every copied data, freelist, branch, leaf, and overflow page is written to
  the same page id it had in the source snapshot
- the destination logical size is the pinned `page_count`

Backup must not renumber reachable pages into a dense layout. Any such rewrite
would require pointer rewriting and must be treated as compaction.

### Non-Copied Pages Below `page_count`

Some page ids below the pinned `page_count` may be neither reachable live pages
nor referenced by the visible freelist state. The backup output must still be
deterministic and must not accidentally create valid extra state at those page
ids.

M7 should implement this by writing deterministic invalid bytes for every
non-copied page below the pinned `page_count`. Accepted approaches:

- zero-fill the page, if zero bytes cannot validate as any supported page type
- write a fixed invalid page header pattern plus zero padding

The chosen invalid fill must be documented next to the encoder and reused
consistently in tests.

### Pages Beyond `page_count`

Backup must not copy bytes beyond the pinned `page_count`, including ignored
tail pages left by failed commits. The output file must therefore end at the
logical size implied by the pinned snapshot rather than the source file's
physical length if the source contains extra trailing bytes.

## Output Metadata Rules

### One-Valid-Metadata Requirement

The backup output must contain exactly one valid metadata slot for the pinned
snapshot. The other metadata slot must be intentionally invalid so the output
cannot present:

- two equal valid generations
- an older still-valid inactive source slot
- ambiguous source history copied into the backup

This rule applies even when both source metadata slots are valid.

### Selected Metadata Contents

The valid destination metadata slot must describe the same logical snapshot that
the read transaction pinned:

- same selected root page id
- same selected freelist root page id
- same `page_count`
- same visible `txn_id`
- same visible `generation`, unless implementation-local metadata encoding
  requires a documented equivalent that still preserves M0 selection rules

Header and metadata duplicated fields must remain mutually consistent.

### Invalid Metadata Slot Strategy

The simplest required strategy is:

1. write the pinned metadata contents into one canonical destination metadata
   slot
2. leave the other destination metadata slot unwritten if the file starts from
   deterministic invalid bytes, or write an explicitly invalid form

The implementation must not copy the inactive source metadata slot verbatim if
it would still validate in the destination.

## Destination Creation, Overwrite, And Temporary-File Rules

### New Destination Rules

When the destination path does not exist:

1. create a temporary file in the same parent directory as the final path
2. write and optionally sync the full backup image into that temporary file
3. sync the parent directory when creating a new file, where the platform
   supports directory sync
4. rename the temporary file into place
5. sync the parent directory again after rename when required for crash safety

Using the same parent directory is required so rename behavior matches the final
filesystem and stays as atomic as the platform allows.

### `overwrite = false`

When `overwrite = false`, backup must fail before writing if the destination
path already exists. It must not truncate, replace, or partially overwrite an
existing file.

### `overwrite = true`

When `overwrite = true`, backup must still write through a temporary file first.
The existing destination remains untouched until the new backup image has:

- completed all page writes
- written its single valid metadata slot
- completed requested file sync
- completed any required pre-rename directory sync

Only then may the implementation rename the temporary file over the destination
using platform-appropriate replace semantics.

### Temporary Name Rules

The temporary file naming scheme must be explicit and easy to diagnose. The
recommended shape is:

`<destination>.tmp.<random-or-unique-suffix>`

Requirements:

- the temp file lives in the destination's parent directory
- the temp file name must not collide predictably across concurrent attempts
- error messages must include both destination and temp path when useful

### Failure Cleanup Rules

On failure, the source database must remain unchanged and the implementation
must attempt best-effort cleanup of the temporary file.

Cleanup policy:

- remove the temp file when it is clearly safe to do so
- if removal fails, return the primary backup failure and include temp path
  context
- never delete or truncate the pre-existing destination file on a failed
  overwrite path

## Concrete Implementation Steps

1. Reuse the normal read-transaction path to pin metadata and reader state.
2. Reuse M6 walkers to enumerate live tree pages, overflow chains, freelist
   pages, `reusable`, and `pending_free`.
3. Build an in-memory page ownership bitmap or set for the pinned `page_count`.
4. Create a temp destination file in the target directory.
5. Write the immutable header from the source format fields.
6. For each page id from the first non-header page through pinned
   `page_count - 1`, either:
   - copy the pinned source page bytes to the same page id, or
   - write deterministic invalid bytes for non-copied page ids.
7. Write exactly one valid metadata slot for the pinned snapshot.
8. Ensure the other metadata slot remains invalid.
9. Sync the destination file when `sync_output = true`.
10. Sync the parent directory for create/rename phases where supported.
11. Rename the temp file into place according to `overwrite`.
12. Close the read transaction and release any reader slot.
13. In tests, reopen the destination or run `verify()` before treating backup as
    successful.

## Test Plan

### Snapshot Correctness

- backup from an idle database reproduces the current visible state
- backup started before a writer commit preserves the older pinned state
- reopened backup does not include keys or metadata from later commits
- long-running reader backup remains stable across multiple later commits

### Page Layout And Metadata

- backup preserves original page ids for root, leaf, branch, freelist, and
  overflow pages
- backup file length matches pinned `page_count` rather than source tail length
- backup excludes ignored tail pages from failed commits
- backup output contains exactly one valid metadata slot
- the valid metadata slot matches the pinned snapshot's root, freelist,
  `txn_id`, `generation`, and `page_count`
- the second metadata slot is invalid and cannot be selected by `open()`

### Freelist And Pending-Free Preservation

- backup of a database with `reusable` pages preserves those references exactly
- backup of a database with `pending_free` groups preserves those groups without
  promoting them
- reopened backup can perform later writes that reuse only pages the preserved
  snapshot made safely reusable

### Overflow And Bucket Coverage

- backup preserves large values stored through overflow chains
- backup preserves bucket catalog contents and bucket iteration order
- backup preserves branch-to-child and leaf-to-overflow references exactly

### Overwrite And Temp-File Behavior

- backup to an existing path with `overwrite = false` fails before writing
- backup with `overwrite = true` replaces an existing valid backup only after
  the new backup is fully written
- temp file is created in the destination directory
- failed overwrite never corrupts the previously valid destination file

### Failure Handling

- destination create failure leaves source unchanged and releases the read tx
- mid-copy write failure leaves source unchanged and no active reader slot
- destination sync failure leaves source unchanged and reports phase context
- temp-file cleanup failure is reported without hiding the primary write or sync
  failure
- `ENOSPC` or short write does not produce a falsely trusted destination

### Validation

- every successful backup reopens through normal `open()`
- every successful backup passes `verify()`
- backup of a source with one invalid inactive metadata slot still produces one
  valid destination metadata slot

## Acceptance Criteria

- Backup always starts from a pinned read transaction snapshot.
- Later writer commits never appear in a backup already in progress.
- Backup output preserves page ids rather than rewriting them.
- Backup preserves visible bucket, overflow, freelist, `reusable`, and
  `pending_free` state from the pinned snapshot.
- Backup output contains exactly one valid metadata slot and one invalid slot.
- Backup never copies ignored tail pages beyond the pinned `page_count`.
- `overwrite = false` fails before writing when the destination already exists.
- `overwrite = true` uses a same-directory temporary file and never performs
  destructive in-place overwrite.
- Source state remains unchanged on every backup failure path.
- A successful backup opens normally and passes `verify()` on all supported
  platforms.
