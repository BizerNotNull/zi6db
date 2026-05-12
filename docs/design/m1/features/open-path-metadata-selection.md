# M1 Feature Plan: Open Path Metadata Selection

This plan defines the concrete M1 implementation work for selecting the current metadata state during `open()`. It refines the M1 implementation plan without changing the frozen M0 recovery rules.

## 1. Scope

This feature covers only startup validation and metadata selection for an existing database file. It does not implement transaction execution, commit writes, page reuse, repair, migration, or crash fault injection.

The implementation must:

- read and validate the immutable header before interpreting page-sized structures
- validate metadata A and metadata B independently
- select the current state by metadata `generation`
- reject ambiguous or internally inconsistent states
- validate the selected root and freelist pages before `open()` succeeds
- tolerate unreachable tail residue without adding it to the freelist
- map validation failures to the M0 public error categories

## 2. Inputs And Outputs

Inputs:

- database path
- open options, including `read_only`, `lock_mode`, durability mode, and expected page-size policy if exposed
- file bytes at the path

Outputs on success:

- opened database handle
- decoded immutable header fields
- selected metadata snapshot
- selected metadata slot, `A` or `B`
- complete whole-page file length
- recorded tail-residue facts for diagnostics only

Outputs on failure:

- no published database state
- acquired file handles and guards released
- public error category from the M0 boundary contract

## 3. Open Order

`open()` must execute in this order:

1. Normalize the path enough for process-local guard lookup.
2. Acquire the requested M1 guard before publishing a handle.
3. Open the file using the requested read-only or read-write mode.
4. Read exactly the fixed `512` byte header bootstrap prefix from offset `0`.
5. Validate the header bootstrap and decode `page_size`.
6. Read the file length and compute complete whole pages as `floor(file_len / page_size)`.
7. Record `tail_residue_bytes = file_len % page_size`.
8. Read metadata page A from physical page `1`.
9. Read metadata page B from physical page `2`.
10. Validate each metadata candidate independently against the validated header and complete whole-page length.
11. Discard invalid candidates, preserving internal reasons for diagnostics and tests.
12. Select between remaining candidates using the A/B selection rules.
13. Publish the selected metadata snapshot and selected slot into the in-memory database state.

No metadata, root, or freelist page may be decoded until the header has fully validated. The header must never be repaired or inferred from metadata.

## 4. File Length And Tail Residue

The open path must distinguish three file-length concepts:

| Concept | Rule | Use |
| --- | --- | --- |
| `file_len` | Raw byte length from the filesystem | Diagnostics and complete-page calculation |
| `complete_pages` | `file_len / page_size`, rounded down | Upper bound for addressable pages |
| `tail_residue_bytes` | `file_len % page_size` | Unreachable bytes after the last complete page |

Tail residue rules:

- if `file_len < 512`, return `FormatError`
- if the header validates but pages required by a metadata candidate are not complete, that candidate is invalid
- if the selected metadata `page_count > complete_pages`, return `Corruption`
- complete pages beyond selected `page_count` are ignored
- partial bytes beyond the last complete page are ignored
- ignored tail pages and bytes are not inserted into the freelist
- ignored tail pages and bytes do not make `open()` fail by themselves

The implementation should expose tail residue only as internal diagnostics. It must not become part of the committed database state.

## 5. Header Precondition

Metadata validation can start only after the header validator has accepted:

- header magic and recognizable bootstrap shape
- format major and minor compatibility
- required feature flags
- little-endian marker
- page-size bounds
- required-zero reserved fields
- header checksum

The decoded header values become the only trusted source for page size, format version, and feature compatibility during this `open()` call. Conflicts between header and metadata duplicate fields are corruption.

## 6. Metadata Candidate Validation

Validate A and B with the same function:

```text
validate_metadata_candidate(header, physical_page_id, bytes, complete_pages) -> Candidate
```

The candidate result should include:

- `valid`
- `slot`
- `generation`
- `txn_id`
- `page_count`
- `root_page_id`
- `freelist_page_id`
- internal invalid reason when rejected

Validation order for each candidate:

1. Confirm the physical page exists as a complete page.
2. Decode the metadata page without trusting any field yet.
3. Validate metadata magic.
4. Validate page type is `META`.
5. Validate `slot_id` matches the physical page: page `1` is A and page `2` is B.
6. Validate metadata page size equals the header page size.
7. Validate metadata format major and minor are compatible with the header.
8. Validate metadata required feature flags are supported and match the header contract.
9. Validate duplicated immutable header fields match the validated header.
10. Validate required-zero reserved fields.
11. Validate the metadata checksum.
12. Validate `page_count >= 5` for the M1 initial layout.
13. Validate `page_count <= complete_pages`.
14. Validate `root_page_id >= FIRST_DATA_PAGE_ID`.
15. Validate `freelist_page_id >= FIRST_DATA_PAGE_ID`.
16. Validate `root_page_id < page_count`.
17. Validate `freelist_page_id < page_count`.
18. Validate the referenced root page.
19. Validate the referenced freelist page.

A validation failure invalidates only that candidate unless the failure is an I/O failure reading required bytes. Candidate invalidation becomes public `Corruption` only after selection determines no acceptable state exists or the two surviving candidates are ambiguous or non-monotonic.

## 7. Critical Page Validation

Root and freelist validation are part of metadata candidate validity. They are not later tie-breakers.

For the root page referenced by a candidate:

- read the complete page by positional I/O
- validate normal-page checksum coverage
- validate `page_id` equals `root_page_id`
- validate page type is the frozen empty-root leaf type expected by M1
- validate page generation is compatible with the metadata generation rules defined by the format
- validate required-zero reserved fields
- validate empty leaf structure constraints used by M1

For the freelist page referenced by a candidate:

- read the complete page by positional I/O
- validate normal-page checksum coverage
- validate `page_id` equals `freelist_page_id`
- validate page type is `FREELIST`
- validate page generation is compatible with the metadata generation rules defined by the format
- validate required-zero reserved fields
- validate zero reusable pages
- validate zero pending-free groups
- reject any non-empty freelist state in M1 unless a later milestone explicitly extends this path

If a metadata candidate points to an out-of-range, wrong-type, or checksum-invalid critical page, discard that candidate.

## 8. A/B Selection

After independent validation:

| Candidate State | Result |
| --- | --- |
| A invalid, B invalid | `Corruption` |
| A valid, B invalid | select A |
| A invalid, B valid | select B |
| A valid, B valid, A generation greater than B | select A after monotonicity check |
| A valid, B valid, B generation greater than A | select B after monotonicity check |
| A valid, B valid, equal generation | `Corruption` |

Selection must use only `generation` as the ordering field. `txn_id` is a consistency field, not a tie-breaker.

The selected slot must be stored in database state so the M3 commit path can write the inactive slot without rediscovering state.

## 9. Transaction Monotonicity

When both metadata candidates are valid and generations differ:

- identify the older and newer candidate by `generation`
- if `newer.txn_id < older.txn_id`, return `Corruption`
- otherwise select the newer candidate

M1 should also reject obviously impossible single-candidate metadata, such as `generation == 0` or `txn_id == 0`, because M0 initialization starts at `generation = 1` and `txn_id = 1`.

M1 must not select by `txn_id`, even when the higher `txn_id` belongs to the lower generation. That state is corruption, not an alternate ordering signal.

## 10. Error Mapping

Public lifecycle errors must be stable even if internal validators preserve richer reasons.

| Condition | Public Error |
| --- | --- |
| file shorter than the header bootstrap prefix | `FormatError` |
| path contains no recognizable database bootstrap | `FormatError` |
| recognizable header with bad endianness, reserved fields, page size, or checksum | `Corruption` |
| unsupported format major, future unsupported minor, or unsupported required feature flag | `IncompatibleVersion` |
| metadata read fails because required bytes cannot be read | `IoError` for OS failure, otherwise candidate invalid |
| metadata checksum, page type, slot id, reserved field, duplicate-field, or range failure | candidate invalid; final public error is `Corruption` if no valid selection remains |
| both metadata candidates invalid | `Corruption` |
| equal metadata generations | `Corruption` |
| newer generation has lower `txn_id` | `Corruption` |
| selected `page_count` exceeds complete whole pages in the file | `Corruption` |
| selected root or freelist page validation fails | `Corruption` |
| required positional read, write, sync, or close operation fails | `IoError` |
| requested lock semantics cannot be guaranteed | `LockConflict` |
| lifecycle misuse during open or close | `InvalidState` |

Internal names such as `ChecksumMismatch`, `SlotMismatch`, `PageOutOfRange`, or `TailResiduePresent` must not leak as public lifecycle errors.

## 11. Implementation Steps

1. Add metadata candidate and selected-metadata structs in the format or db layer.
2. Implement a reusable metadata candidate validator that accepts physical page id and validated header.
3. Implement root and freelist page validators that can run before the B+Tree engine exists.
4. Implement file-length accounting with complete pages and tail residue.
5. Implement A/B selection as a pure function over candidate validation results.
6. Implement transaction monotonicity checks in the selector.
7. Wire the selector into `DB.open()` after header validation and before publishing state.
8. Preserve selected slot and metadata snapshot in `DB`.
9. Add table-driven tests for candidate validation, selection, and error mapping.
10. Keep detailed invalid reasons private or test-only while folding public errors at the lifecycle boundary.

## 12. Test Matrix

| Area | Case | Expected Result |
| --- | --- | --- |
| success | fresh database has valid A and invalid B | open succeeds and selects A |
| success | metadata A invalid, metadata B valid | open succeeds and selects B |
| success | A and B valid, A generation higher | open succeeds and selects A |
| success | A and B valid, B generation higher | open succeeds and selects B |
| header | file shorter than `512` bytes | `FormatError` |
| header | recognizable header with bad checksum | `Corruption` |
| header | invalid endian marker | `Corruption` |
| header | unsupported major version | `IncompatibleVersion` |
| header | unsupported required feature flag | `IncompatibleVersion` |
| metadata | both metadata pages invalid | `Corruption` |
| metadata | metadata page has wrong page type | candidate invalid; both invalid means `Corruption` |
| metadata | metadata `slot_id` does not match physical page | candidate invalid; both invalid means `Corruption` |
| metadata | metadata page size differs from header | candidate invalid; both invalid means `Corruption` |
| metadata | metadata duplicate field conflicts with header | candidate invalid; both invalid means `Corruption` |
| metadata | metadata checksum mismatch | candidate invalid; both invalid means `Corruption` |
| selection | valid A and B have equal generation with identical bytes | `Corruption` |
| selection | valid A and B have equal generation with different bytes | `Corruption` |
| selection | newer generation has lower `txn_id` | `Corruption` |
| monotonicity | single candidate has `generation = 0` | `Corruption` |
| monotonicity | single candidate has `txn_id = 0` | `Corruption` |
| tail | selected `page_count` exceeds complete whole pages | `Corruption` |
| tail | complete extra pages beyond selected `page_count` | open succeeds and ignores tail |
| tail | partial bytes after last complete page | open succeeds and ignores tail |
| critical pages | root page id below `FIRST_DATA_PAGE_ID` | candidate invalid; final `Corruption` if selected state cannot validate |
| critical pages | freelist page id below `FIRST_DATA_PAGE_ID` | candidate invalid; final `Corruption` if selected state cannot validate |
| critical pages | root page id out of candidate `page_count` | candidate invalid; final `Corruption` if selected state cannot validate |
| critical pages | freelist page id out of candidate `page_count` | candidate invalid; final `Corruption` if selected state cannot validate |
| critical pages | root page wrong type | candidate invalid; final `Corruption` if selected state cannot validate |
| critical pages | freelist page wrong type | candidate invalid; final `Corruption` if selected state cannot validate |
| critical pages | root checksum mismatch | candidate invalid; final `Corruption` if selected state cannot validate |
| critical pages | freelist checksum mismatch | candidate invalid; final `Corruption` if selected state cannot validate |
| I/O | positional read returns an OS error | `IoError` |
| I/O | short read of a required complete page | `IoError` or documented candidate invalidation; no panic |
| lifecycle | open failure releases guard and file resources | later valid open can proceed |

Test names should describe the user-visible rule, for example `open_selects_newer_metadata_by_generation` and `open_rejects_equal_generation_metadata`.

## 13. Acceptance

This feature is complete when:

- `open()` always validates the `512` byte header before metadata pages
- metadata A and B use one shared validation path
- metadata slot ids must match physical pages `1` and `2`
- metadata/header duplicate-field conflicts are rejected
- root and freelist pages are validated before state is published
- both-invalid metadata returns `Corruption`
- one-valid metadata is accepted regardless of whether it is A or B
- two-valid metadata is ordered only by `generation`
- equal-generation metadata always returns `Corruption`
- decreasing `txn_id` across increasing `generation` returns `Corruption`
- selected `page_count` cannot exceed complete whole pages in the file
- unreachable complete tail pages and partial tail bytes are ignored and never reused
- public errors match the M0 categories
- tests cover success, header failures, candidate invalidation, A/B selection, monotonicity, tail residue, critical pages, I/O failures, and guard cleanup

## 14. Non-Goals

This feature must not:

- repair a corrupted header or metadata page
- infer database state from tail pages
- reclaim unreachable pages into the freelist
- implement commit-time metadata switching
- implement non-empty freelist migration
- implement B+Tree mutation or lookup
- introduce `mmap`, direct I/O, or alternate page-cache behavior
- weaken M0 recovery semantics for convenience
