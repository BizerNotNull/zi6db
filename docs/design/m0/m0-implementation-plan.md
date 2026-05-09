# M0 Implementation Plan

This document is based on `M0: Core Design Freeze` in [MILESTONES.md](../../MILESTONES.md). Its goal is to freeze the core on-disk format, transaction commit path, and recovery rules of `zi6db` before the storage engine is formally implemented, so we can avoid high-cost rework during M1-M4.

## 1. Goals

M0 does not deliver a runnable database. Instead, it delivers a set of design documents and API sketches that are concrete enough to directly guide implementation, validation, and fault-injection testing. After M0 is complete, the team should be able to answer the following questions clearly:

- How a single commit turns dirty pages in memory into a valid state on disk.
- How startup recovery selects the last valid commit.
- How metadata, the root page, the freelist, and transaction IDs transition during commit and rollback.
- What write-ordering and `fsync` requirements apply on Linux, macOS, and Windows.
- When `commit()` is allowed to return success, and what transaction state remains after failure.
- How future versions determine whether a file format is compatible or must be rejected.

## 2. Design Constraints and Core Invariants

All later storage, transaction, recovery, freelist, and API design must obey the following invariants:

- The current visible state on disk may only be referenced by one valid metadata page.
- A metadata page is the logical switch point of a commit; the system must not assume physical page writes are inherently atomic.
- The system must not rely on torn-write atomicity of metadata pages. A torn metadata write must be treated as invalid when checksum verification fails, `magic` / `format version` does not match, or field self-consistency checks fail.
- Non-metadata dirty pages must be persisted before the metadata page.
- Root pages, freelist pages, and B+Tree pages must all use copy-on-write and must never overwrite the current visible state in place.
- A read transaction may only observe the snapshot corresponding to the metadata generation selected when it began.
- New pages, dirty pages, and freelist updates produced by an uncommitted write transaction must remain invisible to other transactions.
- Rollback only discards in-memory transaction state and does not require repairing the disk.
- The recovery flow may only select metadata that has a correct checksum, a compatible version, self-consistent fields, and the newest ordering field.
- Extra pages at the end of the file, pages unreachable from the current metadata, and new pages from incomplete commits do not constitute a valid database state.
- The role of checksums is to detect torn writes, partial writes, and silent corruption with very high probability. Checksums do not provide automatic repair or cryptographic integrity guarantees.

## 3. Crash and Durability Assumptions

M0 must first freeze the failure model that `zi6db` promises to support, as well as the boundaries it explicitly does not promise to support:

- Supported: process crash.
- Supported: system reboot.
- Supported: power loss at any point during commit.
- Supported: a torn metadata page write being recognized as invalid via checksum mismatch or structural validation failure.
- Supported: incomplete writes to new pages not referenced by the current metadata not being selected by recovery.
- Not supported: a storage device losing data after reporting a successful flush.
- Not supported: checksum collisions, or silent tampering in regions not covered by the checksum by the file system or hardware.
- Not supported: multiple processes writing to the same database file while violating the locking protocol.
- Not supported: designing recovery as a "best effort database repair" flow. The primary goal is to reliably identify the last valid state or return `Corruption`.

## 4. Deliverables

The M0 documentation is split into four required documents under `docs/design/`:

- `storage-format.md`
  Explains page size, file header, metadata pages, normal pages, B+Tree nodes, reserved overflow format, checksum coverage, and version fields.
- `transaction-recovery.md`
  Explains the read/write transaction model, transaction IDs, commit protocol, rollback semantics, dual-metadata-page selection rules, and the startup recovery flow.
- `page-lifecycle-and-commit-ordering.md`
  Explains page allocation, dirty-page ownership, the copy-on-write path, freelist transaction semantics, flush ordering, and platform-specific sync requirements.
- `api-sketch.md`
  Explains `open/create/close`, transaction lifecycle, cursor invalidation rules, and the top-level error model.

These documents must also include four classes of frozen appendices:

- `m0-decision-freeze-table`
  Lists the decisions that must have explicit defaults by M0, and whether later milestones may change them.
- `format-constants`
  Lists constants such as magic, format version, page type, endianness, checksum algorithm, fixed page IDs, and default page size.
- `crash-safety-matrix`
  Lists the on-disk state, API return semantics, and recovery selection result for different crash points, as the basis for future fault-injection cases.
- `crash-and-corruption-test-spec`
  Organizes scenarios such as crash matrix cases, metadata A/B selection, checksum failures, page-type mismatches, `ENOSPC` / short write / sync failure, and version compatibility into a future test specification.

This document only defines the execution plan for M0. It does not replace the design documents themselves.

## 5. M0 Freeze Decision Table

The goal of M0 is not to "list open questions," but to freeze the default decisions that all later implementation must follow. The final design docs must include a decision table like the following, with an explicit default value for every item:

| Area | Decision | M0 must provide a default choice | May M1+ change it? |
| --- | --- | --- | --- |
| Page size | Default, minimum, maximum, alignment rules | Yes | No, unless the format version is upgraded |
| Header bootstrap | Bootstrap prefix size, how `page_size` is read | Yes | No |
| Metadata selection | Whether `generation` or `txn_id` is the primary ordering field | Yes | No |
| Checksum | Algorithm, coverage range, which pages require validation | Yes | No |
| I/O model | Whether the first version uses positional I/O or `mmap` | Yes | No, unless re-reviewed |
| Durability | Exact guarantees of `strict` / `normal` / `unsafe` | Yes | New modes may be added, but existing semantics may not change |
| Locking | Whether M1 rejects cross-process writes, and how M7 extends it | Yes | Can be extended, but promised safety cannot be weakened |
| Commit success point | When `commit()` may return success | Yes | No |
| Commit failure semantics | Transaction state after `fsync` / `FlushFileBuffers` failure | Yes | No |
| Freelist persistence | Whether `pending_free` is persisted and how it transitions after restart | Yes | No, unless the format version is upgraded |

## 6. Work Breakdown

### 6.1 Freeze the Storage Format

The following decisions must be made first:

- Page size, alignment rules, and page ID encoding.
- The bootstrap prefix size of the file header must be fixed and must not depend on parsing `page_size`, for example a fixed first 512B or 4KB.
- Fixed page numbering rules:
  - `page 0`: file header
  - `page 1`: metadata A
  - `page 2`: metadata B
  - `page >= 3`: normal pages
- The file header layout, and the responsibility boundary between the header and metadata pages.
- The header should be as immutable as possible and should only contain bootstrap information such as `magic`, format version, endianness, `page_size`, and feature flags. Dynamic state should not be written into the header.
- Whether the header has a checksum, whether header corruption immediately returns `Corruption`, and which side wins when duplicated fields in the header and metadata conflict must all be fixed in M0.
- Required defaults to freeze immediately:
  - The file header is an immutable bootstrap record.
  - Header corruption directly returns `Corruption`.
  - Metadata is not used to recover a corrupted header.
  - Conflicting duplicated fields between the header and metadata return `Corruption`.
- Metadata page fields: magic, format version, page size, root page, freelist page, transaction ID, generation, page count or `high_watermark`, checksum, and state flags.
- The primary ordering field of metadata must be frozen directly in M0 and must not remain "to be defined."
- Page headers for B+Tree internal / leaf pages, slot layout, and variable-length key/value encoding.
- The type number and reserved fields for overflow pages, so M6 will not need to break format compatibility.
- All pages other than the file header must define a validation-carrying page header or an equivalent fixed validation layout containing at least: `page_id`, `page_type`, checksum, and flags/reserved fields. If `page_generation` or `owning_txn_id` is not physically stored on every page, M0 must state explicitly which page classes omit it and why that omission does not weaken recovery or corruption detection guarantees.
- Checksum coverage, algorithm, byte order, and error classification on validation failure for key structures.
- M0 must freeze which page classes carry checksums and when those checksums are validated during open, recovery, and normal reads. At minimum, metadata, root, freelist, branch, leaf, and overflow pages must either:
  - carry page-level checksums over at least `page_id || page_type || page_body_without_checksum`, or
  - use an explicitly documented equivalent integrity scheme with the same recovery role.
- Validation of non-metadata pages must not remain "implementation-defined" after M0. Any page class not validated on every read path must be called out explicitly with the exact safety tradeoff.
- How a transaction rolls back when file-tail extension fails, and why newly added tail pages are ignored on the next startup when commit fails.
- The first-version I/O model must be frozen as positional read/write. `mmap` must not enter the M1-M4 implementation path. If `mmap` is introduced later, it must not change the on-disk format, commit ordering, or recovery semantics.

Definition of done:

- It is possible to manually draw the logical layout of the database file from the document alone.
- "Format fields" and "runtime-derived state" can be clearly distinguished.
- The startup sequence can be clearly explained as "read fixed bootstrap prefix first, then parse `page_size`, then interpret metadata and normal pages."
- Frozen constants can be written explicitly for direct use in M1.

### 6.2 Freeze the Transaction and Recovery Model

The following rules must be defined:

- Snapshot boundaries for read transactions and write transactions.
- Under the single-writer model, ownership rules for dirty pages and new pages within a write transaction.
- Whether commit uses copy-on-write, and how the root page, freelist, and metadata switch.
- When both `generation` and `txn_id` exist, what each one is responsible for and which one is the primary metadata ordering field.
- Required default freeze: `generation` is the sole primary ordering field for startup recovery. `txn_id` is used for transaction visibility, freelist `pending/reusable` decisions, and observability. Under normal conditions, both increase monotonically together on committed write transactions.
- If the two metadata pages have the same `generation`, both have valid checksums, but their contents differ, return `Corruption`. If `generation` increases but `txn_id` is not monotonic, return `Corruption`.
- The metadata slot selection rule must also be frozen directly in M0:
  - A normal `commit()` always writes to the slot other than the currently selected metadata slot.
  - If the current selection is `META_A`, this commit must write `META_B`; if the current selection is `META_B`, this commit must write `META_A`.
  - When initializing a new database, only one metadata copy must be valid and the other must be zeroed/invalid; do not start with two valid copies sharing the same `generation`.
  - If A/B have the same `generation` but different contents on startup, return `Corruption`. If they have the same `generation` and identical contents, the first version must still return `Corruption` to avoid masking duplicate-write or initialization bugs.
- Validation and selection order for the header, metadata, root, and freelist at startup.
- Recovery must treat root/freelist structural validation as part of metadata validity before final winner selection. A metadata page whose referenced root or freelist fails required validation is invalid and must be discarded before comparing it against another candidate.
- Which in-memory states are discarded on rollback, and which pages do not affect the on-disk state.
- How checksum failure participates in metadata selection, page validation, and error returns.
- Recovery must validate root/freelist page types, checksums, range fields, and `high_watermark` self-consistency before a metadata page can be considered a valid startup candidate.
- The success point of `commit()`, failure points, the transaction state after failure, and the conservative semantics that "a new metadata page may already exist on disk even if a failure was returned."

Definition of done:

- The three paths "successful commit," "crash in the middle of commit," and "rollback before commit" can be described step by step.
- It can be explained why recovery never selects a half-committed state.
- A deterministic startup recovery algorithm can be given instead of only describing comparison principles.

### 6.3 Freeze Page Lifecycle, Freelist, and Commit Ordering

The following details need to be completed:

- The page ID space and fixed page numbering.
- File growth, `high_watermark` / `page_count`, and tail allocation rules.
- Page allocation sources: file-tail extension, freelist reuse, and overflow reservation.
- Page lifecycle: `clean -> cloned -> dirty -> written -> committed/discarded`.
- The two-phase `pending` / `reusable` semantics of the freelist.
- Whether `pending_free` is persisted in the freelist page, and when it may transition to `reusable` after restart.
- Reader pinning rules: deleted pages may not be reused while older read transactions are still active.
- Under copy-on-write, how the root page, freelist page, and metadata page switch together.
- The exact steps of commit, including the two sync boundaries.
- Durability requirements on each platform: Linux `fsync` / `fdatasync`, macOS conservative mode with `F_FULLFSYNC`, and the Windows conservative strategy using `FlushFileBuffers`.
- Directory-entry durability requirements when creating a new database file.
- Locking strategy in single-process and multi-process modes, and the implementation boundary between M1 and M7.

Definition of done:

- The commit ordering can be mapped directly into a later code step list.
- It can be stated explicitly which steps must sync and which only require ordered writes.
- A crash-safety matrix and corruption test spec can be derived directly from the document.

### 6.4 API Lifecycle and Error Model

The minimal developer-facing interface needs to define:

- `DB.open/create/close`
- `beginRead/beginWrite`
- `commit/rollback`
- A minimal placeholder interface for `bucket/table`
- A basic sketch of cursor traversal
- Initial open/create options such as `durability_mode`, `lock_mode`, `read_only`, and `page_size`
- Error categories: format errors, incompatible version, checksum failures, lock conflicts, invalid transaction state, and I/O errors
- Cursor lifetime, invalidation rules after transaction close, whether read transactions may be upgraded, and how `close()` interacts with active transactions

Definition of done:

- The API sketch can support interface evolution from M1 to M5.
- The error model maps one-to-one to recovery, locking, and validation behavior.
- The API section focuses on transaction lifecycle and error semantics instead of freezing too much bucket/table behavior too early.

## 7. Recommended Execution Order

To reduce rework risk, it is recommended to proceed in the following order:

1. Write `storage-format.md` first and freeze all on-disk structures and reserved fields.
2. Then write `transaction-recovery.md` and make metadata switching and recovery rules fully explicit.
3. Next write `page-lifecycle-and-commit-ordering.md` and turn dirty-page handling and flush ordering into executable steps.
4. Finally write `api-sketch.md` to ensure the external API follows the lower-level transaction and format design, rather than pulling the lower layers in the opposite direction.

This order corresponds to the highest-risk dependency chain in `MILESTONES.md`: M3, M4, and M6 all depend on M0 freezing metadata atomicity, commit ordering, and freelist semantics early enough.

## 8. Commit and Recovery Algorithm Requirements

M0 does not need to implement code, but it does need to freeze the commit path and recovery path to a level close to pseudocode.

### 8.1 Commit Pseudocode Requirements

M0 must freeze at least the following level of detail in `page-lifecycle-and-commit-ordering.md`:

1. Allocate a new `txn_id` and `generation`.
2. Write the B+Tree pages, overflow pages, and freelist page modified by this transaction to page IDs that are not currently visible.
3. Perform a data sync on the database file to ensure those non-metadata pages are durably readable.
4. Construct the inactive metadata page:
   - `root_page = new_root`
   - `freelist_page = new_freelist`
   - `txn_id = new_txn_id`
   - `generation = previous_generation + 1`
   - `page_count/high_watermark = new_value`
   - `checksum = checksum(metadata_without_checksum)`
5. Write the inactive metadata page.
6. Sync the database file again.
7. Switch the current metadata in memory only.
8. Release the write transaction lock.

The selection rule for the inactive metadata slot must be fixed as:

- When the currently selected metadata is `META_A`, this `commit()` may only write `META_B`.
- When the currently selected metadata is `META_B`, this `commit()` may only write `META_A`.
- For a newly created database, initialize as: `META_A.generation = 1` and valid, `META_B` zeroed/invalid, `root_page` pointing to an empty root leaf, and `freelist_page` pointing to an empty freelist.

M0 must also freeze API semantics:

- `commit()` may return success only after the inactive metadata page has been written and the required sync has completed.
- `commit()` failure semantics must be frozen into three classes:
  - A. Failure before metadata page write begins:
    - `commit()` returns `IoError`.
    - In-process current metadata does not switch.
    - The write transaction enters a terminal `failed` or `closed` state, and the caller may only discard the transaction object.
    - Recovery must select the old metadata, assuming the old metadata is valid; otherwise return `Corruption`.
  - B. The metadata page has started writing, but metadata sync is not confirmed successful:
    - `commit()` returns `IoError`.
    - In-process current metadata does not switch.
    - The write transaction enters a terminal `failed` or `closed` state, and the caller may only discard the transaction object.
    - Recovery may select the old metadata, or may select the new metadata if its checksum and structural validation both succeed.
    - The caller must not infer from a failed `commit()` return that the transaction definitely did not reach disk.
  - C. Metadata sync is explicitly successful:
    - Only then may `commit()` return success.
    - Recovery must be able to select that metadata unless a later device or file-system violation occurs outside the promised failure model.
- For class B and any case where the result after metadata sync cannot be confirmed, recovery is the only authoritative state selector. The caller must not rely on in-memory state to decide whether the transaction ultimately took effect.

The following conservative platform strategies also need to be stated clearly:

- Linux: define whether the first version uses `fsync()` or `fdatasync()`, and explain metadata durability requirements when the file grows.
- macOS: define whether conservative mode uses `fcntl(F_FULLFSYNC)`.
- Windows: define the conservative strategy of calling `FlushFileBuffers` after writes.
- New database files: define whether syncing the parent directory is additionally required to make the directory entry durable.

`durability_mode` must also specify guarantee levels, rather than only listing mode names:

- `strict`
  - Sync data pages before writing metadata.
  - `commit()` may return success only after metadata sync succeeds.
  - Parent directory sync rules must be defined for new-file and rename/create paths.
- `normal`
  - A lighter-weight sync method such as `fdatasync` may be allowed instead of the most conservative strategy.
  - It still promises that once `commit()` returns success, crash recovery can restore that commit.
- `unsafe`
  - Some or all sync operations may be skipped.
  - It only promises process-crash safety, not system power-loss safety.

If M1 does not yet want to carry the complexity of multiple modes, M0 must still state clearly that the implementation freezes only `strict` semantics, while other modes remain API reservations and are not exposed early.

### 8.2 Startup Recovery Algorithm Requirements

M0 must freeze a deterministic recovery algorithm in `transaction-recovery.md`, for example:

1. Read the fixed-size header bootstrap prefix.
2. Validate `magic`, `format version`, endianness, and `page_size`.
3. Decode `page_size` from the bootstrap prefix, then interpret metadata pages and normal page layout.
4. Read metadata page A and metadata page B.
5. For each metadata page:
   - Validate `magic`
   - Validate that `page_size` matches the header
   - Validate format-version compatibility
   - Validate checksum
   - Validate the ranges of `root_page`, `freelist_page`, and `high_watermark/page_count`
   - Validate the page type and checksum of the referenced `root` / `freelist` pages
6. Discard all invalid metadata pages.
7. If both are invalid, return `Corruption`.
8. If only one is valid, select it.
9. If both are valid, select the newer one by the primary ordering field.
10. If the primary ordering field is equal but the contents differ, return `Corruption`.
11. If the primary ordering field is equal and the contents are fully identical, the first version must still return `Corruption` to avoid masking duplicate-write or initialization bugs.

M0 must make the following explicit:

- The fixed size of the header bootstrap prefix, and whether the header is immutable.
- Header corruption directly returns `Corruption`, and recovery from metadata is not allowed.
- Conflicts in duplicated fields between the header and metadata return `Corruption`.
- Whether `generation` and `txn_id` both exist.
- Which one is the primary ordering field for metadata selection.
- Whether the other field is used as an auxiliary consistency check or only for user-visible semantics.

## 9. Crash-Safety Matrix and Test-Spec Requirements

M0 must add a crash matrix covering at least the following scenarios:

- Crash before data-page writes.
- Crash after partial data-page writes.
- Crash after data-page sync but before metadata write.
- Failure before metadata page write begins.
- Torn metadata page write.
- Crash after metadata write but before sync.
- Crash after metadata sync.
- Metadata sync returns an error or its result cannot be confirmed.

Each scenario must clearly state:

- What content may remain on disk.
- Whether `commit()` may already have returned success.
- Whether recovery should select the old state or the new state.
- Whether the outcome is successful recovery or `Corruption`.

One allowed but easily misunderstood case must be written especially clearly here:

- If metadata is fully written but a crash happens before sync, recovery may observe the new metadata as valid.
- This does not violate the semantics, because before `commit()` returns success, it is allowed that the transaction "may already have reached disk."
- But once `commit()` returns success, that transaction must satisfy the persistence promise of the corresponding durability mode.

In addition to the crash matrix, M0 must directly produce a test specification covering at least:

- Metadata A/B selection cases.
- Checksum mismatch cases, plus an assertion that "checksum mismatch must make the page invalid; checksum collision is outside the promised failure model."
- `high_watermark` out-of-range cases.
- `root_page` / `freelist_page` page-type mismatch cases.
- `ENOSPC` / short write / sync failure cases.
- Format-version compatibility cases.

## 10. Recommended Freeze Items

The following items must be frozen directly in M0 to avoid later reverse changes to the disk format after M1 begins:

- Fixed page IDs: `HEADER_PAGE_ID`, `META_A_PAGE_ID`, `META_B_PAGE_ID`, `FIRST_DATA_PAGE_ID`
- Format constants: `MAGIC`, `FORMAT_MAJOR`, `FORMAT_MINOR`, `MIN_PAGE_SIZE`, `MAX_PAGE_SIZE`
- Page types: `PAGE_TYPE_META`, `PAGE_TYPE_FREELIST`, `PAGE_TYPE_BRANCH`, `PAGE_TYPE_LEAF`, `PAGE_TYPE_OVERFLOW`
- Encoding rules: endianness, page-header alignment, slot encoding rules
- Header bootstrap: fixed prefix size, whether it has a checksum, header immutability rules
- Checksum: algorithm, coverage range, which page classes carry checksums, when they are validated, and failure semantics
- Metadata fields: `generation`, `txn_id`, `root_page`, `freelist_page`, `page_count/high_watermark`
- First-version I/O model: frozen as positional read/write
- The success point and failure semantics of `commit()`
- Guarantee levels of `durability_mode`

## 11. Freelist and Locking Semantics Requirements

M0 must freeze freelist and locking semantics to a level sufficient to support M3, M6, and M7.

### 11.1 Freelist Semantics

- Pages deleted by a write transaction do not immediately enter the reusable set.
- Deleted pages first enter `pending_free[txn_id]`.
- The freelist page must persist both `reusable` pages and `pending_free` entries; `pending_free` must be grouped at least by `freeing_txn_id`.
- `pending_free` may move into the `reusable freelist` only after all read transactions older than the deleting `txn_id` have ended.
- The current write transaction must not reuse pages it just freed if they may still be referenced by an older root.
- On rollback, in-memory records of `pending_free` updates and newly allocated pages are discarded directly.
- During recovery, only the freelist state reachable from the current metadata is trusted. Space must not be implicitly reclaimed just because leftover pages exist at the end of the file.
- In M1 single-process mode, there are no active readers after process restart. Therefore, after startup recovery, all pending pages that are not pinned by current read transactions and satisfy the generational rule may move to reusable.
- In M7 multi-process mode, a reader lock table or equivalent mechanism must be used to detect stale readers and compute `oldest_reader_txn_id` to decide which pages may be reused.

### 11.2 Phased Goals for the Locking Strategy

Minimum goal for M1:

- Multiple read transactions are allowed within the same process.
- At most one write transaction is allowed within the same process.
- Write transactions and read transactions may run concurrently, and read transactions read an older metadata snapshot.
- Cross-process write mode may remain unsupported for now, or opening may be rejected directly.

Goal for M7:

- Cross-process shared read lock.
- Cross-process exclusive write lock.
- After metadata switches, new read transactions obtain the new snapshot by rereading metadata.
- Stale readers participate in freelist page reuse decisions through transaction generations.

## 12. API Lifecycle and Error-Semantics Sketch

`api-sketch.md` does not need to be the final API in M0, but it should freeze the following constraints early:

- `DB.open(path, options)` / `DB.create(path, options)` / `DB.close()`
- `options` should consider at least:
  - `create_if_missing`
  - `read_only`
  - `page_size`
  - `durability_mode: strict | normal | unsafe`
  - `lock_mode: single_process | multi_process`
  - `max_read_transactions`
- `Transaction` should consider at least:
  - `id()`
  - `is_read_only()`
  - `commit()`
  - `rollback()`
  - `get_bucket()`
  - `create_bucket()`
  - `delete_bucket()`
- `Cursor` should consider at least:
  - `first()`
  - `last()`
  - `seek(key)`
  - `next()`
  - `prev()`
  - `key()`
  - `value()`
- The error model should cover at least:
  - `FormatError`
  - `IncompatibleVersion`
  - `ChecksumMismatch`
  - `Corruption`
  - `LockConflict`
  - `TransactionClosed`
  - `ReadOnlyTransaction`
  - `WriteConflict`
  - `IoError`

The following should also be made explicit:

- Cursor lifetime is bound to the transaction.
- All cursors become invalid once the transaction is closed.
- Read transactions do not support implicit upgrade to write transactions unless M0 explicitly chooses to support it.
- A write transaction cannot continue to be used after `commit()` or `rollback()`.
- When `close()` encounters active transactions, it should either return an error or follow an explicit shutdown policy.
- The exact relationship among `commit()` returning success, `commit()` returning failure, and process crash.
- What terminal transaction state is entered when `fsync` / `fdatasync` / `FlushFileBuffers` fails.

## 13. Acceptance Checklist

When M0 is complete, it is recommended to verify the following one by one:

- Whether the disk write order of one successful commit can be described precisely.
- Whether startup selection of the valid state from two metadata copies can be described precisely.
- Whether the state transitions of the root, freelist, and dirty pages during commit and rollback are clearly defined.
- Whether conservative sync strategies are defined for Linux, macOS, and Windows.
- Whether the exact semantics among `commit()` success, `commit()` failure, and process crash are defined.
- Whether transaction state after `fsync` / `fdatasync` / `FlushFileBuffers` failure is defined.
- Whether the disk format version strategy and the reject-on-open behavior for incompatible versions are defined.
- Whether the header bootstrap read rule and the handling rule for conflicts between the header and metadata are defined.
- Whether stable format hooks are reserved for future overflow-page and freelist implementation needs.
- Whether the metadata selection algorithm and the relationship between `generation` and `txn_id` are frozen.
- Whether the checksum algorithm, coverage range, and failure semantics are frozen.
- Whether it is frozen whether page checksum covers `page_id`, `page_type`, and `page_generation`.
- Whether the page ID space, fixed page numbering, and `high_watermark/page_count` rules are frozen.
- Whether the freelist `pending -> reusable` semantics, persistence method, and restart-time migration rules are frozen.
- Whether crash-safety guarantees for each `durability_mode` level are defined.
- Whether parent-directory sync rules are defined for new-file and rename/create paths.
- Whether a crash-safety matrix and corruption test spec have been produced that can map directly to fault-injection testing.
- Whether an API sketch and error model sufficient for M1 implementation kickoff have been produced.

## 14. Non-Goals

The following should not be expanded into implementation work during M0:

- Full runnable B+Tree insertion/deletion algorithm code.
- A full freelist reclamation implementation.
- Final cross-platform file-locking implementation details.
- Performance tuning, benchmark conclusions, or advanced configuration options.
- `mmap`, direct I/O, or page-cache strategy optimization itself is not completed in M0. If introduced later, it still must not change the on-disk format, commit ordering, or recovery semantics frozen by M0.

These items may be implemented in later milestones, but M0 must first ensure that the related format and semantics are reserved in advance and do not contradict each other.
