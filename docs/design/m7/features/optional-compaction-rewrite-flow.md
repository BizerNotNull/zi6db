# Optional Compaction Rewrite Flow

This document defines the implementation plan for the optional M7 compaction
rewrite flow. It refines the M7 milestone requirement that compaction, if
shipped, must be a destination-only rewrite first and an optional source
replacement second. It also assumes the M6 handoff seams already exist for
live-page classification, overflow validation, freelist validation, and
transaction-pinned snapshot traversal.

Compaction is not a repair path. It must preserve the selected logical
snapshot, reject corrupted inputs, and avoid mutating the source database in
place.

## Preconditions

Compaction may only ship after all of the following are true:

- `verify()` is stable enough to validate both source snapshots and compacted
  outputs with the same page validators used by startup recovery.
- Backup/snapshot export already passes cross-platform tests for pinned
  snapshots, metadata output, sync behavior, and reopen verification.
- M6 live-page classification can enumerate without overlap:
  - fixed pages
  - reachable branch and leaf pages
  - bucket root references
  - reachable overflow chains
  - freelist pages
  - reusable pages
  - `pending_free` groups
  - ignored tail pages outside selected `page_count`
- A read-only walker can traverse the selected root, freelist chain, bucket
  roots, and overflow chains while recording ownership and duplication errors.
- A page rewrite layer exists for all pointer-bearing structures that compaction
  needs to rewrite:
  - metadata root and freelist pointers
  - branch child page IDs
  - bucket root references stored in leaf values
  - leaf overflow references
  - overflow next-page links
  - freelist chain links
- Cross-process locking is complete enough to guarantee that optional source
  replacement can demand exclusive ownership before rename or replacement.

Compaction must be deferred if any precondition above is incomplete. M7 should
ship backup and verify without compaction rather than weaken safety boundaries.

## Destination-Only Rewrite

The first shippable compaction mode is destination-only:

```zig
pub const CompactOptions = struct {
    sync_output: bool = true,
    replace_source: bool = false,
};

pub fn compact(
    db: *DB,
    destination_path: []const u8,
    options: CompactOptions,
) !CompactStats;
```

Required behavior:

- Start from a pinned read transaction snapshot.
- Run source validation using the normal metadata selection path plus
  verifier-compatible traversal of all live structures required by the pinned
  snapshot.
- Fail before writing a trusted output if the source snapshot is corrupt,
  ambiguous, or cannot be traversed completely.
- Create a new temporary destination file and write the compacted image there.
- Rewrite only the selected live snapshot:
  - fixed header page
  - reachable tree pages
  - reachable overflow pages
  - the compacted freelist representation
  - one valid metadata slot
- Exclude from the compacted output:
  - reusable source pages
  - source `pending_free` pages
  - ignored tail pages
  - unreachable failed-commit leftovers
- Produce a dense page layout beginning at page `3`, except for any pages
  needed by the compacted file's own freelist representation.
- Preserve logical content exactly:
  - bucket names
  - key ordering
  - key/value bytes
  - overflow value bytes
  - transaction-visible snapshot contents
- Write one valid metadata slot and leave the other metadata slot invalid.
- Sync the destination file when `sync_output = true`.
- Sync the parent directory for newly created destination files where
  supported.
- Close the pinned read transaction and temporary resources on every success or
  failure path.

Destination-only compaction is the correctness baseline. Source replacement is
not allowed until this mode is stable and fully tested.

## Remapping

Compaction rewrites live pages into new dense page IDs, so it needs an explicit
remapping phase rather than ad hoc pointer patching.

### Remapping Model

The implementation should build a complete `old_page_id -> new_page_id`
translation table for every live page in the pinned snapshot before metadata is
finalized. The table must cover:

- branch pages
- leaf pages
- overflow pages
- freelist pages that remain necessary in the compacted output

Fixed pages `0`, `1`, and `2` keep their reserved identities.

### Rewrite Order

Recommended order:

1. Traverse the selected snapshot and classify live pages.
2. Allocate compacted destination page IDs densely.
3. Build the remap table for all live pages.
4. Rewrite page bodies into destination pages while patching internal
   references through the remap table.
5. Build the compacted freelist image for the destination file.
6. Write destination metadata that points only at remapped destination roots.

### Structures That Must Be Patched

Every old page reference encoded in the live snapshot must be remapped:

- branch child pointers
- branch fence or child-bound structures if they embed page IDs
- bucket root references encoded inside leaf payloads
- leaf overflow-reference records:
  - `first_overflow_page_id`
  - `overflow_page_count`
  - `logical_value_length` remains unchanged
- overflow page `next_overflow_page_id`
- freelist chain `next_freelist_page_id`
- metadata `root_page_id`
- metadata `freelist_page_id`

The rewrite path must not "mostly copy" pages and patch only some fields.
Compaction succeeds only when all pointer-bearing encodings are rewritten from
the same authoritative remap table.

### Destination Freelist Rules

The compacted output should normally contain no reusable or pending source
pages. The destination freelist should therefore be either:

- empty, using the engine's canonical empty-freelist representation, or
- minimally populated only if the destination format requires explicit freelist
  pages to represent emptiness

Source `pending_free` and source `reusable` entries must not be copied as live
free space into the compacted output. Compaction is a logical snapshot rewrite,
not a preservation of historical free-space accounting.

## Verify-Before-Replace

Compaction must have a hard verification boundary before any source replacement
is attempted.

Required sequence:

1. Build the compacted destination file at a temporary path.
2. Flush it according to `sync_output`.
3. Run `verify()` against the destination file using the same validator set
   expected for normal reopen.
4. Optionally reopen the destination in tests and perform logical content
   checks.
5. Only after successful verification may the implementation consider source
   replacement when `replace_source = true`.

Failure rules:

- If destination verification fails, compaction returns failure and the source
  file remains untouched.
- A destination file that fails verification must not be treated as a valid
  backup or replacement candidate.
- Verification findings from the compacted output should report destination path
  context and page IDs in the rewritten layout.

This boundary keeps compaction from becoming a best-effort repair path. A
corrupt source fails early; a malformed rewrite fails before replacement.

## Crash Boundaries

Compaction must document crash behavior by phase.

### Source Snapshot Phase

- Crash before the pinned read snapshot is established: no compaction artifact
  is trusted, source remains authoritative.
- Crash after snapshot begin but before destination metadata write: source
  remains authoritative; any partial destination is ignored or cleaned up by
  the caller.

### Destination Build Phase

- Crash during destination page writes: source remains valid and unchanged.
- Crash after destination page writes but before destination metadata write:
  source remains valid; destination is incomplete and must not be trusted.
- Crash after destination metadata write but before destination file sync:
  source remains valid; destination may be absent, invalid, or valid depending
  on platform persistence, but it must never affect source correctness.
- Crash after destination file sync but before directory sync: source remains
  valid; destination visibility depends on platform rename and directory
  persistence rules.

### Replacement Phase

Optional source replacement is a separate crash boundary:

- Crash before replacement begins: original source path remains authoritative.
- Crash during replacement: behavior must match documented platform rename
  semantics and must be covered by tests.
- Crash after replacement rename but before directory sync: final path may be
  old or new depending on platform guarantees; M7 must document this precisely
  and only support replacement where the behavior is acceptable.

If crash-safe replacement cannot be documented cleanly for a platform, M7 must
expose destination-only compaction there and reject `replace_source = true`.

## Deferability

Compaction is explicitly deferrable within M7.

M7 should defer this feature when any of the following are true:

- verify issue kinds, traversal semantics, or JSON output are still unstable
- backup has unresolved correctness or crash-boundary issues
- live-page classification cannot prove non-overlap between tree, overflow, and
  freelist ownership
- remapping support is incomplete for bucket roots, overflow chains, or
  freelist chains
- replacement rename rules are not yet reliable on Linux, macOS, and Windows
- test coverage exists only for happy paths and not for corruption, crash, and
  partial-write boundaries

If deferred, M7 should document:

- destination-only backup remains supported
- file shrinking remains unavailable
- normal delete/update behavior still reuses pages inside the source file but
  does not reduce physical file size
- compaction remains a post-M7 hardening task rather than an implicit guarantee

## Tests

The compaction test plan must cover correctness, remapping, crash boundaries,
and replacement gating.

### Core Rewrite Tests

- compacted copy opens successfully
- compacted copy passes `verify()`
- compacted copy preserves bucket names, key order, and value bytes
- compacted copy preserves large overflow values across reopen
- compacted copy excludes source reusable pages, source pending pages, ignored
  tail pages, and unreachable failed-commit leftovers
- compacted copy uses dense remapped page IDs
- compacted copy emits exactly one valid metadata slot

### Remapping Tests

- branch child pointers reference remapped destination pages only
- bucket root references stored in leaf payloads are remapped correctly
- overflow-reference records point to remapped overflow chains
- overflow `next_overflow_page_id` links follow the remapped chain exactly
- freelist chain links, if present in the compacted output, are remapped
  correctly
- metadata root and freelist pointers reference destination page IDs only
- no stale source page ID remains in any rewritten pointer-bearing structure

### Source Rejection Tests

- corrupted source metadata causes compaction failure before trusted output
- corrupted live branch or leaf page causes compaction failure
- corrupted overflow chain causes compaction failure
- corrupted selected freelist structure causes compaction failure
- equal-generation metadata conflict causes compaction failure rather than
  arbitrary selection

### Crash And Failure Tests

- destination `ENOSPC` leaves source valid and unchanged
- short write during destination build leaves no trusted compacted file
- crash before destination metadata write leaves source authoritative
- crash after destination metadata write but before sync leaves source
  authoritative
- crash during destination-only compaction before replacement request leaves
  source authoritative
- verification failure after rewrite prevents replacement

### Replacement Tests

- `replace_source = false` never mutates the source path
- `replace_source = true` requires exclusive ownership and fails cleanly when
  readers or other processes still hold the database open
- replacement uses a temporary path plus platform-appropriate rename rules
- crash during replacement follows the documented platform result
- reopened replacement target preserves logical content and passes `verify()`

## Acceptance

This feature is complete only when all of the following are true:

- Compaction is implemented as a destination-only rewrite first, not in-place
  page surgery.
- Source validation rejects corruption and metadata ambiguity before a trusted
  compacted file is produced.
- Remapping covers metadata, branch, bucket, overflow, and freelist references
  consistently from one authoritative translation table.
- The compacted output preserves logical contents exactly while excluding source
  reusable pages, source pending pages, ignored tail pages, and unreachable
  failed-commit leftovers.
- The compacted output contains a valid header, one valid metadata slot, and a
  verifier-clean rewritten page graph.
- `verify()` runs successfully on the destination before any optional source
  replacement begins.
- Crash boundaries are documented for destination build and replacement phases.
- Replacement is either crash-safe and tested on a platform, or explicitly
  unsupported there.
- The feature can be deferred cleanly without changing M6 or M7 backup
  semantics.
- Tests cover clean rewrites, remapping correctness, corruption rejection,
  `ENOSPC` and short writes, crash boundaries, and exclusive replacement rules.
