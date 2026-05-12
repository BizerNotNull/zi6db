# M4 Implementation Plan: Crash Safety And Recovery

## Goal

M4 makes `zi6db` recover to a valid committed state after process interruption, I/O failure, torn metadata writes, or critical metadata corruption.

This milestone implements the durability and recovery rules frozen in M0 without changing the on-disk format. Metadata remains the only commit switch point, pages remain copy-on-write, and startup recovery is the only authority after an ambiguous commit failure.

## Inputs And Dependencies

M4 depends on the M1-M3 implementation exposing these stable pieces:

- Positional whole-page read and write helpers.
- Header and metadata encoders/decoders for the M0 format.
- B+Tree root and freelist page encoders with page checksums.
- Single writer transaction ownership and rollback terminal-state behavior.
- M3 commit hooks for non-metadata page writes, pre-metadata sync, metadata write, metadata sync, and in-memory metadata publication.
- A recorded base metadata slot on each write transaction, so M4 can choose the inactive slot without re-running recovery heuristics mid-commit.
- In-memory current metadata snapshot used by readers and writers.

M4 must follow these M0 design documents:

- `docs/design/m0/storage-format.md`
- `docs/design/m0/transaction-recovery.md`
- `docs/design/m0/page-lifecycle-and-commit-ordering.md`
- `docs/design/m0/crash-safety-matrix.md`
- `docs/design/m0/crash-and-corruption-test-spec.md`
- `docs/design/m0/format-constants.md`

## Scope

M4 includes:

- Crash-safe commit protocol with strict write and sync ordering.
- Dual metadata recovery using metadata slots A and B.
- CRC32C checksum validation for header, metadata, and critical normal pages.
- Startup validation that rejects invalid headers, incompatible formats, invalid metadata, and invalid selected root or freelist pages.
- Fault injection around page writes, metadata writes, file growth, and sync boundaries.
- Configurable durability mode behavior for `strict`, `normal`, and `unsafe`.
- Internal checker path for metadata, page type, checksum, and reachable-page sanity.
- Tests that exercise the M0 crash and corruption matrix.

## Non-Goals

M4 does not include:

- Bucket or cursor API expansion; that belongs to M5.
- Overflow value support beyond validating reserved page type rules; full support belongs to M6.
- Full freelist reuse optimization beyond preserving persisted `reusable` and `pending_free` semantics needed for safety.
- Multi-process file locking hardening; that belongs to M7.
- Backup, compaction, vacuum, or user-facing integrity CLI; these belong to M7.
- Salvaging or repairing corrupted pages.
- Inferring reusable pages from unreachable tail pages.
- Changing page size, metadata layout, checksum algorithm, or format version rules frozen by M0.

## Implementation Principles

- Never overwrite a visible root, freelist, branch, leaf, or overflow page in place.
- Treat metadata pages as the only durable commit switch.
- Validate referenced root and freelist pages before selecting metadata.
- Return `Corruption` for supported-format files whose required structures are invalid.
- Return `IncompatibleVersion` for recognized files requiring unsupported format versions or required feature flags.
- Keep commit failure terminal for the write transaction.
- Do not switch the in-memory current metadata pointer until the selected durability mode has completed its required post-metadata persistence boundary.
- Treat `generation` as the only primary metadata ordering field; `txn_id` is a monotonic consistency and visibility check, never a tie-breaker.
- Never select metadata by scanning roots, freelists, tail pages, file size, write timestamp, or slot preference.
- Treat root and freelist page validation failures as metadata-slot invalidation before comparing generations.
- Preserve M0 Class B semantics: after a metadata write starts but metadata sync is not confirmed, the returned failure is ambiguous on disk and reopening is the only authority.

## Crash-Safe Commit Protocol

The M4 commit path should be implemented as a small state machine so tests can inject faults before and after every durable boundary.

### Commit Steps

1. Acquire the writer commit path exclusively.
2. Read the writer base metadata snapshot.
3. Reserve `new_txn_id = base.txn_id + 1`.
4. Reserve `new_generation = base.generation + 1`.
5. Promote only reader-safe pending-free groups into `reusable`; preserve all non-eligible `pending_free` groups.
6. Apply the committing transaction's frees under `pending_free[new_txn_id]`; do not reuse pages freed by the same transaction.
7. Finalize all dirty branch, leaf, and overflow pages.
8. Finalize the new freelist page with persisted `reusable` and `pending_free` groups.
9. Compute normal-page CRC32C checksums after all page fields are final.
10. Write every non-metadata page using positional whole-page writes.
11. Apply the pre-metadata sync boundary required by the selected durability mode.
12. Select the inactive metadata slot from the writer's base selected slot: A commits to B, and B commits to A.
13. Build metadata with the new root page id, freelist page id, page count, generation, transaction id, selected slot id, and state flags.
14. Compute the metadata CRC32C checksum over the full metadata page except the checksum field.
15. Write the inactive metadata page using one positional whole-page write.
16. Apply the metadata sync boundary required by the selected durability mode.
17. Switch the in-memory current metadata pointer to the new metadata.
18. Mark the transaction closed and release the writer lock.

### Commit Ordering Invariants

- No non-metadata page write may target a page id reachable from the writer base metadata.
- The new metadata page must not be built until the final root page id, freelist page id, page count, generation, and transaction id are known.
- The metadata checksum must be the last metadata field finalized before the metadata page write.
- In-memory metadata publication is a separate phase after the metadata sync boundary, even for `unsafe`.
- Commit success means both the API durability promise and in-memory publication have completed.
- Commit failure before in-memory publication leaves new readers on the old metadata snapshot, even if a later reopen may select the new metadata.
- Commit failure after in-memory publication is not a valid M4 state; the implementation must order operations so no fallible persistence step remains after publication.

### Failure Handling

Failures before step 17 must leave the in-memory current metadata unchanged. The write transaction must enter a terminal failed state and the writer lock must be released only after cleanup that cannot publish the new metadata.

| Failure class | Example | Commit result | Required next-open behavior |
| --- | --- | --- | --- |
| Class A: before metadata write begins | short data-page write, file growth failure, pre-metadata sync failure | `IoError`; transaction terminal | old metadata selected if old metadata is valid |
| Class B: metadata write may have started, metadata sync not confirmed | short metadata write, torn metadata write, crash after full metadata write before sync, sync returns error after metadata write | `IoError` when the process observes the failure; transaction terminal | recovery may select old or new metadata, but only by deterministic validation and generation comparison |
| Class C: metadata sync confirmed | all required writes and syncs completed | success only after in-memory metadata publication | new metadata selected within the promised failure model |

Class B is intentionally ambiguous. Callers and tests must not interpret an `IoError` returned after the metadata write phase starts as proof that the transaction did not commit on disk. The current process must continue to expose the old metadata after that failure, while a later `open()` may expose the new metadata if the new slot validates and has the higher generation.

## Dual Metadata Recovery

Startup recovery must be deterministic and must not use heuristics beyond the M0 rules.

### Recovery Steps

1. Read the first `512` bytes of the file.
2. Validate header magic, format version, endianness, page size, required feature flags, reserved bits, and header checksum.
3. Decode page size from the header.
4. Read metadata slot A from page `1`.
5. Read metadata slot B from page `2`.
6. Validate each metadata page independently.
7. Discard invalid metadata pages.
8. If both metadata pages are invalid, return `Corruption`.
9. If exactly one metadata page is valid, select it.
10. If both metadata pages are valid and generations differ, select the larger generation.
11. If both metadata pages are valid and generations are equal, return `Corruption`.
12. When both pages validate and generations differ, return `Corruption` if the larger generation has a lower `txn_id` than the older valid slot.
13. Publish the selected metadata as the current in-memory database state.

### Selection Guardrails

- Slot A is valid only when its on-disk `slot_id` says A and it was read from page `1`.
- Slot B is valid only when its on-disk `slot_id` says B and it was read from page `2`.
- A valid checksum is necessary but not sufficient; range checks and critical page validation must also pass before a slot can participate in selection.
- If one slot is invalid because it points at a bad root or freelist, discard that slot before comparing generations.
- If the newer-looking slot by generation is invalid, select the older valid slot; do not return `Corruption` solely because the invalid slot has a larger raw generation field.
- If both slots validate and have equal generations, return `Corruption` even when the bytes are identical.
- Do not use `txn_id`, physical slot order, page write order, or file length as a fallback selector when generations tie.

### Metadata Validation

A metadata page is valid only if all checks pass:

- Metadata magic matches `META_MAGIC`.
- Page type is `PAGE_TYPE_META`.
- Slot id matches the physical slot being read.
- Page size matches the header.
- Format major and minor are supported.
- Required feature flags are supported.
- Reserved fields required to be zero are zero.
- Checksum matches.
- `page_count >= 4`.
- `root_page_id >= 3` and `root_page_id < page_count`.
- `freelist_page_id >= 3` and `freelist_page_id < page_count`.
- Referenced root page has a valid checksum and page type `PAGE_TYPE_LEAF` or `PAGE_TYPE_BRANCH`.
- Referenced freelist page has a valid checksum and page type `PAGE_TYPE_FREELIST`.

### Monotonicity Checks

If both metadata pages validate:

- The selected page must have the larger generation.
- The newer generation must not have a lower `txn_id` than the older generation.
- Equal generations always produce `Corruption`, even if every field is identical.
- The expected normal case is `generation = older.generation + 1` and `txn_id = older.txn_id + 1`, but recovery should only require monotonic non-decrease of `txn_id` relative to generation because it may open files produced by future compatible minors.

## Checksum Validation

M4 should centralize checksum behavior in a reusable validation module rather than scattering ad hoc checks through open, commit, and checker code.

### Required Checksum Coverage

| Structure | Algorithm | Coverage |
| --- | --- | --- |
| Header bootstrap prefix | CRC32C | Header fields except the header checksum field |
| Metadata page | CRC32C | Full metadata page except the metadata checksum field |
| Normal page | CRC32C | Common page header and body except the page checksum field |

### Validation Modes

| Mode | Used by | Behavior |
| --- | --- | --- |
| Decode validation | `open()` and page reads on critical paths | Return structured validation failure |
| Commit finalization | write transaction commit | Compute checksums after final page layout |
| Checker validation | internal verify path | Collect all failures without stopping at the first non-fatal page |
| Test mutation validation | corruption tests | Assert the expected checksum failure maps to `Corruption` at public boundaries |

## Sync Policy

M4 must implement the API-visible durability modes without weakening the M0 contract.

| Mode | Pre-metadata sync | Metadata sync | Success promise |
| --- | --- | --- | --- |
| `strict` | Required | Required | Power-loss-safe within the platform failure model |
| `normal` | Required initially | Required initially | Same as `strict` until a proven lighter path is documented |
| `unsafe` | Optional | Optional | No power-loss durability promise; no in-process partial visibility before success |

Mode-specific rules:

- `strict` must not return commit success until both sync boundaries have succeeded and the in-memory metadata pointer has switched.
- `normal` may use a lighter primitive only after the implementation documents why the success promise is still crash-recoverable for file growth and metadata replacement on that platform; until then it must alias `strict`.
- `unsafe` may skip durable flushes, but it must still preserve logical ordering in the process: write referenced pages before metadata, write metadata before publication, and never expose the new metadata in memory before `commit()` returns success.
- All modes must keep Class A, Class B, and Class C failure semantics identical at the API boundary.
- A mode may weaken persistence after power loss only where the table says so; it may not weaken checksum validation, metadata selection, copy-on-write, transaction terminal-state behavior, or in-process snapshot isolation.

### Platform Mapping

| Platform | `strict` implementation | `normal` initial implementation | `unsafe` implementation |
| --- | --- | --- | --- |
| Linux | `fsync()` | Alias of `strict` until `fdatasync()` safety is proven | Skip durable flushes |
| macOS | `fsync()` plus `F_FULLFSYNC` where available | `fsync()` without `F_FULLFSYNC` may be allowed after tests; otherwise alias `strict` | Skip durable flushes |
| Windows | `FlushFileBuffers()` | Alias of `strict` | Skip durable flushes |

For new database creation in `strict`, file content must be flushed and the parent directory entry must be synced where the platform exposes a reliable directory sync path.

## Fault Injection Plan

M4 should introduce a test-only fault injection layer beneath the storage engine and above OS file calls.

### Injection Points

| Injection point | Faults to simulate | Expected use |
| --- | --- | --- |
| File tail growth | `ENOSPC`, short extension, write failure | Confirm old metadata remains authoritative |
| Non-metadata page write before first byte | write error | Confirm commit fails before publication |
| Non-metadata page write after partial bytes | short write, torn page | Confirm old metadata remains selected |
| Pre-metadata sync | sync error | Confirm commit fails and old metadata remains selected |
| Metadata page write before first byte | write error | Confirm old metadata remains selected |
| Metadata page write after partial bytes | short write, torn page | Confirm recovery rejects invalid metadata |
| Metadata sync | sync error | Confirm commit fails and reopen decides state |
| Metadata checksum mutation | bit flip | Confirm invalid slot is discarded |
| Root/freelist checksum mutation | bit flip | Confirm referenced metadata becomes invalid |

### Harness Requirements

- Each injected failure must record the last completed commit phase.
- Tests must reopen the database through normal startup recovery after simulated crashes.
- Tests must assert both the returned commit result and the recovered metadata generation.
- Tests for Class B must also assert that the pre-reopen process did not switch in-memory metadata after `commit()` returned `IoError`.
- Tests must assert the selected metadata slot as well as the selected generation whenever A/B slot behavior is relevant.
- The harness must support deterministic byte mutation for metadata, root, freelist, and arbitrary normal pages.
- The harness must support deterministic short writes at byte offsets `0`, middle-of-page, and last-byte-minus-one for both metadata and normal pages.
- The harness must simulate sync calls that fail before issuing a flush, fail after the OS reports an uncertain result, and succeed.
- The harness must simulate crashes after successful writes but before the next configured sync boundary.
- Fault injection must be compiled out or unreachable in production builds.

### Required Phase Names

Use stable phase names in the fault harness so crash tests and checker assertions remain readable:

| Phase | Meaning |
| --- | --- |
| `reserve_tail_pages` | File tail growth or page id reservation for this commit |
| `write_data_pages` | Branch, leaf, overflow, and freelist pages are being written |
| `sync_data_pages` | Pre-metadata sync boundary |
| `build_metadata` | Inactive metadata contents and checksum are finalized in memory |
| `write_metadata` | Inactive metadata slot write has started |
| `sync_metadata` | Post-metadata sync boundary |
| `publish_metadata` | In-memory current metadata pointer switches |

Fault injection must be able to stop before and after each phase. No injected fault after `publish_metadata` may represent a persistence failure for the same commit, because M4 must not leave fallible commit I/O after publication.

## Checker Path

M4 adds an internal read-only checker used by tests and future M7 tooling. It can remain non-public in M4.

### Checker Responsibilities

- Validate header bootstrap fields and checksum.
- Validate both metadata slots and report per-slot status.
- Re-run metadata selection rules.
- Validate selected root page type and checksum.
- Validate selected freelist page type and checksum.
- Traverse the selected B+Tree from the selected root.
- Confirm reachable page ids are within `page_count`.
- Confirm reachable page types are allowed for their location.
- Confirm no reachable page id is duplicated in the selected tree traversal.
- Confirm selected freelist entries are within range and do not include fixed pages `0`, `1`, or `2`.
- Report unreachable pages above `page_count` or outside selected metadata as informational, not corruption.
- Distinguish unreachable non-critical pages from critical pages that are referenced but invalid or unreadable.
- Preserve per-slot metadata validation details even when the database as a whole can open using the other slot.

### Checker Result Shape

The checker should return structured findings:

| Severity | Meaning |
| --- | --- |
| `ok` | No issue |
| `info` | Benign condition, such as ignored tail pages after failed commit |
| `warning` | Suspicious but not yet public API failure |
| `corruption` | Public open should fail with `Corruption` |
| `incompatible` | Public open should fail with `IncompatibleVersion` |

Checker findings should include a stable classification code in addition to severity:

| Classification | Severity | Public boundary mapping |
| --- | --- | --- |
| `invalid_header` | `corruption` | `Corruption` |
| `incompatible_format` | `incompatible` | `IncompatibleVersion` |
| `invalid_metadata_slot` | `warning` or `corruption` | `Corruption` only when no valid slot remains or selection rules fail |
| `metadata_checksum_mismatch` | `warning` or `corruption` | Same as `invalid_metadata_slot` |
| `metadata_equal_generation` | `corruption` | `Corruption` |
| `metadata_txn_regression` | `corruption` | `Corruption` |
| `critical_page_out_of_range` | `corruption` | `Corruption` for the metadata slot that references it |
| `critical_page_checksum_mismatch` | `corruption` | `Corruption` for the metadata slot that references it |
| `invalid_page_type` | `corruption` | `Corruption` when the page is selected root, selected freelist, or reachable tree content |
| `duplicate_reachable_page` | `corruption` | `Corruption` |
| `freelist_reserved_page` | `corruption` | `Corruption` |
| `ignored_unreachable_page` | `info` | no public open failure |

For per-slot findings, the checker must record whether the finding invalidates only slot A, only slot B, or the selected database state. A corrupted inactive slot is reportable, but it must not make `open()` fail if the other slot validates and wins under M0 selection rules.

## Execution Order

1. Implement shared CRC32C helpers and checksum field exclusion rules.
2. Add structured page validation results for header, metadata, and normal pages.
3. Implement dual metadata validation and deterministic recovery selection.
4. Wire recovery into `open()` before any transaction state is published.
5. Implement inactive-slot metadata construction from the writer base slot and strict slot alternation in commit.
6. Add the commit phase state machine with explicit phase names.
7. Add platform sync wrappers and durability mode dispatch.
8. Ensure commit failure marks the write transaction terminal and leaves in-memory metadata unchanged.
9. Add the fault injection file adapter for tests.
10. Add crash matrix tests that reopen the database after each injected failure.
11. Add corruption tests for header, metadata, root page, and freelist page mutations.
12. Add the internal checker path and use it in recovery tests for detailed assertions.
13. Document any platform-specific sync limitations discovered during implementation.

## Test Matrix

### Recovery Selection Tests

| Case | Setup | Expected result |
| --- | --- | --- |
| Only A valid | A checksum valid, B zeroed | Select A |
| Only B valid | A checksum invalid, B valid | Select B |
| A newer | A generation greater than B | Select A |
| B newer | B generation greater than A | Select B |
| Both invalid | A and B fail validation | `Corruption` |
| Equal generation, different bytes | A and B valid but same generation | `Corruption` |
| Equal generation, identical bytes | A copied to B | `Corruption` |
| Newer generation with lower txn id | New slot generation increases, txn id decreases | `Corruption` |

### Commit Crash Tests

| Fault point | Commit result | Reopen expectation |
| --- | --- | --- |
| Before any non-metadata page write | `IoError` or simulated crash | Old generation selected |
| During non-metadata page write | `IoError` or simulated crash | Old generation selected |
| After non-metadata writes before sync | simulated crash | Old generation selected |
| Pre-metadata sync failure | `IoError` | Old generation selected |
| During metadata write with torn page | `IoError` or simulated crash | Old or new only if checksum-valid; otherwise old |
| After metadata write before metadata sync | simulated crash | Old or new allowed if validation is deterministic |
| Metadata sync failure | `IoError`; in-memory metadata remains old | Reopen decides old or new |
| After metadata sync success | success | New generation selected |

### Commit Failure Class Tests

| Case | Required assertion |
| --- | --- |
| Class A failure | `commit()` returns `IoError`, transaction is terminal, current process keeps old generation, reopen selects old generation |
| Class B write failure | `commit()` returns `IoError`, transaction is terminal, current process keeps old generation, reopen selects only a valid slot |
| Class B sync uncertainty | `commit()` returns `IoError`, transaction is terminal, current process keeps old generation, reopen may select old or new by validation |
| Class C success | `commit()` returns success only after metadata sync and in-memory publication, reopen selects new generation |

### Corruption Tests

| Mutation | Expected result |
| --- | --- |
| Header magic mismatch | `Corruption` |
| Header checksum mismatch | `Corruption` |
| Unsupported major version | `IncompatibleVersion` |
| Unsupported required feature flag | `IncompatibleVersion` |
| Metadata magic mismatch | Invalid slot; `Corruption` if no other slot valid |
| Metadata checksum mismatch | Invalid slot; `Corruption` if no other slot valid |
| Root page id out of range | Invalid metadata |
| Freelist page id out of range | Invalid metadata |
| Root wrong page type | Invalid metadata |
| Freelist wrong page type | Invalid metadata |
| Root checksum mismatch | Invalid metadata |
| Freelist checksum mismatch | Invalid metadata |

### Sync Policy Tests

| Mode | Test |
| --- | --- |
| `strict` | Verify both sync boundaries are called on commit success |
| `strict` | Verify sync failure prevents in-memory metadata switch |
| `normal` | Verify initial behavior matches documented implementation |
| `unsafe` | Verify metadata is not visible in memory before commit returns success |
| `unsafe` | Verify data-before-metadata write ordering is still used even when sync calls are skipped |
| Create strict | Verify file flush and best-effort parent directory sync behavior |

### Checker Tests

| Case | Expected checker finding |
| --- | --- |
| Clean database | `ok` |
| Ignored tail page after failed commit | `info` |
| Metadata checksum mismatch in inactive older slot while active slot valid | Per-slot corruption, selected state ok |
| Selected root checksum mismatch | `corruption` |
| Selected freelist includes fixed page | `corruption` |
| Reachable page id duplicated in tree traversal | `corruption` |
| Reachable page id greater than or equal to `page_count` | `corruption` |

## Acceptance Checklist

- `commit()` writes and syncs non-metadata pages before publishing metadata in `strict`.
- `commit()` writes the inactive metadata slot only after all referenced pages are finalized.
- `commit()` returns success only after the required metadata sync succeeds for the selected durability mode.
- Commit failures before in-memory publication keep the previous metadata snapshot active.
- Startup validates the header before reading page-sized structures.
- Startup validates both metadata slots independently.
- Startup discards invalid metadata slots before generation comparison.
- Startup rejects equal-generation metadata slots.
- Startup selects the valid metadata slot with the highest generation.
- Startup rejects newer metadata whose `txn_id` goes backwards relative to the older valid slot.
- Critical metadata, root page, and freelist page checksum corruption are detected.
- Fault injection covers data writes, metadata writes, file growth, and both sync boundaries.
- Fault injection covers Class A, Class B, and Class C commit outcomes, including the Class B rule that in-process metadata remains old while reopen may choose old or new.
- Crash simulation tests reopen through normal `open()` instead of test-only recovery shortcuts.
- The checker can distinguish invalid metadata, invalid page types, checksum failures, and unreachable non-critical pages.
- The checker reports per-slot metadata failures separately from selected-state corruption.
- `strict`, `normal`, and `unsafe` sync policies are documented in code comments or API docs where implemented.
- M4 does not change the M0 on-disk format or claim M5-M7 features.
