# M1 Implementation Plan

This document is based on `M1: Single-File Storage Skeleton` in [MILESTONES.md](../../../MILESTONES.md) and the frozen M0 design documents under [docs/design/m0/](../m0/README.md). Its goal is to turn the M0 format and lifecycle rules into the first runnable storage layer for `zi6db`: a database file that can be created, opened, validated, and closed safely.

M1 is intentionally narrow. It does not try to ship a usable KV engine yet. Instead, it establishes the file-format bootstrap path, page I/O foundation, metadata selection logic, and early safety guards that later milestones depend on.

## 1. Goals

After M1 is complete, the implementation should be able to do the following reliably:

- create a brand-new database file with the frozen header and metadata layout
- reopen an existing database file and select the current valid metadata state
- reject invalid headers, corrupted critical pages, and unsupported format versions with clear errors
- read and write fixed-size pages using positional I/O
- close the database cleanly while enforcing the M0 transaction lifecycle rules
- prevent unsafe early multi-process writer usage through conservative guard behavior

## 2. Scope Boundary

M1 implements only the minimum storage skeleton needed for later engine work.

### 2.1 In Scope

- `create/open/close` database lifecycle
- immutable header creation and validation
- metadata A/B page initialization, validation, and selection
- empty root page and empty freelist page initialization
- fixed-size page read/write helpers
- basic file sizing and whole-page offset calculations
- file length checks against selected `page_count`, while tolerating unreachable tail pages
- M0-compatible error mapping for format, corruption, version, lock, and I/O failures
- single-process lifecycle enforcement and early multi-process misuse prevention
- tests for create, reopen, invalid-file rejection, and critical page validation

### 2.2 Explicitly Out Of Scope

- `get/put/delete/exists`
- real read/write transaction execution
- B+Tree mutation logic
- freelist reuse logic beyond initializing and validating an empty freelist page
- restart-time migration of non-empty `pending_free` groups into `reusable`
- crash-fault injection coverage from M4
- large-value overflow behavior from M6
- production-grade cross-process locking from M7

## 3. M0 Constraints That M1 Must Preserve

M1 is not allowed to reinterpret M0. The following rules are already frozen and must be reflected directly in the first implementation:

- page `0` is the immutable file header
- page `1` is metadata slot A
- page `2` is metadata slot B
- page `3+` are normal pages
- default page size is `4096`
- header bootstrap prefix is `512` bytes
- page I/O is positional and page-aligned
- all on-disk integers use little-endian encoding
- header, metadata, and normal-page integrity use `CRC32C`
- normal-page checksums cover `page_id`, `page_type`, `page_generation`, flags, and page body while excluding the checksum field itself
- `generation` is the only metadata ordering field during open
- `txn_id` must remain monotonic with `generation` and is an auxiliary consistency field during open
- the file header cannot be repaired from metadata
- conflicting duplicated header/metadata fields are corruption, not tie-breakers
- only one metadata slot is valid on initial database creation
- metadata slot B must be zeroed or otherwise invalid on initial creation
- each metadata page's `slot_id` must match its physical page (`A` on page `1`, `B` on page `2`)
- metadata `page_size`, format version, and required feature fields must agree with the header before a candidate can be valid
- open must validate the selected root and freelist pages before accepting metadata
- selected metadata must not claim a `page_count` beyond the complete whole pages available in the file
- extra bytes or whole pages beyond selected `page_count` are allowed after failed growth, but M1 must not infer them into the freelist
- `normal` may be implemented as an alias of `strict` in M1
- `unsafe` may exist as a reserved option, but M1 must not imply power-loss durability when syncs are skipped
- cross-process write safety may be conservative, but promised safety must not be weaker than M0
- `mmap`, direct I/O, or page-cache optimization must not enter the M1 storage path

## 4. Deliverables

M1 should produce the following implementation outcomes:

- a minimal `DB` type with `create`, `open`, and `close`
- minimal open/create option handling for `read_only`, `lock_mode`, `page_size`, and the early M1 durability surface
- on-disk encoding and decoding for the header, metadata page, empty root page, and empty freelist page
- page I/O helpers for reading and writing a whole page by page id
- open-path validation that follows the M0 recovery rules for header and metadata selection
- file-size validation that rejects truncated selected state without treating unreachable tail bytes or pages as reusable
- a minimal guard path that rejects clearly unsafe concurrent writer scenarios
- automated tests that cover the M1 exit criteria in `MILESTONES.md`

M1 should also produce developer-facing implementation notes in the same PR that document the local commands used to build and test the new code.

The milestone should not broaden the public API beyond what the storage skeleton needs. Any exported type introduced in M1 should either be part of the M0 API sketch or be clearly internal to the implementation.

## 5. Recommended Source Layout

The repository does not yet have production Zig code, so M1 should establish a layout that can scale into M2-M4 without forcing a refactor. A reasonable first cut is:

- `src/db.zig`
  Database lifecycle entry points and top-level state.
- `src/format/constants.zig`
  Frozen page ids, magic values, page types, and size limits from M0.
- `src/format/header.zig`
  Header encode/decode and validation helpers.
- `src/format/meta.zig`
  Metadata page encode/decode, checksum, and selection helpers.
- `src/format/page.zig`
  Common page header definitions and page offset helpers.
- `src/storage/file.zig`
  Positional file I/O, sync wrappers, file size checks, and create/open primitives.
- `src/storage/lock.zig`
  M1 guard behavior for single-process safety and conservative cross-process rejection.
- `test/`
  Open/create/close and invalid-file tests, or module-adjacent `*_test.zig` files if that fits the code better.

The exact file names may vary, but the responsibilities should stay separated along these boundaries.

### 5.1 Module Boundaries

M1 should keep byte-format code, operating-system I/O code, and database lifecycle code separate enough that later milestones can extend each layer without changing the others.

| Boundary | Owns | Must Not Own | Later Consumers |
| --- | --- | --- | --- |
| `format` | constants, binary layout, checksums, page ids, validation helpers | file handles, locks, allocator ownership policy, public lifecycle decisions | M2 page codecs, M4 recovery, M6 overflow/freelist |
| `storage` | positional reads/writes, file sizing, flush/sync wrappers, directory sync attempts | metadata selection policy, B+Tree semantics, transaction state | M3 commit path, M4 fault injection |
| `db` | open/create/close state machine, options, selected metadata snapshot, guard acquisition/release | raw byte layout details beyond calling codecs, low-level OS syscalls | M2 engine entry points, M3 transactions |
| `lock` | process-local handle registry and pre-M7 lock-mode enforcement | transaction visibility, file format validation | M3 writer exclusivity, M7 file locking |
| `tests` | behavior-driven create/open/corruption coverage and test fixtures | production-only recovery shortcuts or alternate format definitions | M4 crash tests, M7 corruption tests |

Format codecs should expose validation results that preserve detail internally, such as checksum mismatch versus range mismatch, while the public API can fold them into `Corruption` where M0 requires that behavior.

## 6. Work Breakdown

### 6.1 Freeze Runtime Constants In Code

The first implementation step should copy the M0 constants into code in one place and treat them as the only authoritative runtime values:

- header magic and metadata magic
- format major/minor version
- page ids for header, metadata A, metadata B, and first data page
- page types
- default, min, and max page size
- header bootstrap size
- checksum algorithm selection

Definition of done:

- format constants can be imported without circular dependencies
- create and open both use the same constant definitions
- tests can assert exact byte-level layout assumptions without duplicating magic numbers
- magic byte values are chosen once in M1 code and treated as stable emitted format values after the first file is created

### 6.2 Implement Header Encode/Decode And Validation

M1 should implement a single header codec responsible for:

- writing the immutable bootstrap header during `create()`
- reading the first `512` bytes during `open()`
- validating magic, endianness, page size, required-zero reserved fields, feature flags, and checksum
- mapping failures to `Corruption` or `IncompatibleVersion` according to the M0 rules

This step must also freeze the in-code rule that metadata is never consulted to recover a corrupted header.

Recommended validation order:

1. read exactly `HEADER_BOOTSTRAP_PREFIX_SIZE` bytes from offset `0`
2. check enough magic/shape to distinguish a non-database file from a recognized but corrupt `zi6db` file
3. validate format major/minor and required feature flags
4. validate little-endian marker, page-size bounds, and required-zero reserved fields
5. validate the header checksum
6. return the decoded page size only after every bootstrap check has passed

Definition of done:

- a created file contains a stable header with the frozen bootstrap fields
- open can reject truncated, malformed, or checksum-invalid headers deterministically
- page size is known before any metadata page is interpreted

### 6.3 Implement Metadata, Empty Root, And Empty Freelist Initialization

`create()` should initialize the smallest valid database state compatible with M0:

- header page at page `0`
- valid metadata A at page `1`
- invalid or zeroed metadata B at page `2`
- empty root leaf page at page `3`
- empty freelist page at page `4`

The initial metadata should point to:

- `root_page_id = 3`
- `freelist_page_id = 4`
- `page_count = 5`
- `generation = 1`
- `txn_id = 1`

The empty root and freelist pages do not need full M2 behavior yet, but they must already:

- use the common normal-page header
- carry the correct page type
- carry a valid checksum
- preserve the frozen empty on-disk shape expected by later milestones
- encode zero entries in the root leaf without adding search, split, or mutation logic
- encode an empty freelist state that is still structurally compatible with persisted `reusable` and `pending_free` sections
- encode zero reusable pages, zero pending-free groups, and required-zero reserved freelist fields
- satisfy the validation rules that `open()` will apply later

Definition of done:

- a fresh file is internally self-consistent on first open
- metadata B is not accidentally valid with the same generation as metadata A
- future milestones can reuse the initialized root and freelist structures without changing the on-disk format
- M1 does not create, migrate, or reuse non-empty freelist entries because no M1 operation can delete pages

### 6.4 Implement Page I/O Helpers

M1 should introduce positional whole-page I/O utilities that all later milestones can reuse:

- `page_offset(page_id, page_size) -> u64`
- `read_exact_page(file, page_id, buffer)`
- `write_exact_page(file, page_id, buffer)`
- file-size validation for requested page reads
- file-length validation that distinguishes a truncated selected state from harmless unreachable tail bytes or pages
- arithmetic overflow checks for `page_id * page_size`
- zero-fill or explicit initialization for new pages where required

These helpers should be deliberately strict:

- only whole-page reads and writes
- explicit short-read and short-write detection
- clear `IoError` mapping
- no `mmap`
- no treating partial trailing bytes as addressable pages

Definition of done:

- create and open use the same page-offset calculation
- invalid page reads cannot silently succeed on truncated files
- selected metadata whose `page_count` exceeds the file's complete page count is rejected
- complete pages or partial trailing bytes beyond selected `page_count` are ignored for recovery and are not inserted into the freelist
- later milestones can build commit ordering on top of the same I/O layer
- tests can inject or simulate short reads, short writes, and sync failures without replacing format code

### 6.5 Implement The Open Path

The open path is the core of M1. It should follow the M0 recovery order closely, even though M1 is not yet implementing real commits:

1. open the file with the requested options
2. read and validate the `512`-byte header bootstrap
3. decode `page_size`
4. compute the complete whole-page file length and record any partial trailing bytes as unreachable tail residue
5. read metadata A and metadata B
6. validate each metadata page independently, including physical slot id, metadata magic, page type, page size, version, required feature flags, reserved fields, checksum, and header/metadata duplicate-field agreement
7. validate `page_count`, `root_page_id`, and `freelist_page_id` ranges against both M0 limits and the complete whole-page file length
8. validate the referenced root page and freelist page for each metadata candidate
9. discard invalid metadata candidates
10. if both metadata candidates are invalid, return `Corruption`
11. if exactly one metadata candidate is valid, select it
12. if both metadata candidates are valid and their generations are equal, return `Corruption`
13. if both metadata candidates are valid and the newer-by-`generation` candidate has a lower `txn_id`, return `Corruption`
14. select the valid metadata using `generation`
15. publish the selected metadata and selected slot into in-memory database state

M1 must treat the following as public-contract behavior, not internal best effort:

- unsupported format version returns `IncompatibleVersion`
- unsupported required feature flags return `IncompatibleVersion`
- checksum mismatch on critical structures leads to `Corruption`
- wrong page type for root or freelist leads to `Corruption`
- metadata `slot_id` mismatch with the physical metadata page leads to `Corruption`
- metadata/header duplicate-field disagreement leads to `Corruption`
- newer generation with decreasing `txn_id` leads to `Corruption`
- out-of-range `page_count`, `root_page_id`, or `freelist_page_id` leads to `Corruption`
- selected `page_count` beyond the file's complete whole-page length leads to `Corruption`
- complete pages or partial trailing bytes beyond selected `page_count` are ignored and do not make open fail by themselves

Definition of done:

- a valid fresh file reopens successfully
- invalid metadata A can fall back to valid metadata B when appropriate
- both-invalid metadata fails deterministically
- non-monotonic `txn_id` is rejected when both metadata candidates would otherwise validate
- equal-generation metadata copies are rejected
- a selected slot is recorded so M3/M4 can write the inactive slot without rediscovering it

### 6.6 Implement Close Semantics And Early Guard Behavior

M1 does not need the full M3 transaction system, but it does need to respect the M0 lifecycle boundaries now.

At minimum:

- `close()` must release file handles and any in-process lock state
- `close()` must reject closure if active transactions exist once transactions are introduced
- `read_only` must prevent any future write-oriented path from being used
- `lock_mode = .single_process` should enforce one database handle per file path inside the current process, or another equally conservative in-process rule
- if `lock_mode = .multi_process` is not yet truly implemented, `open()` should fail fast with `LockConflict` or `InvalidState` rather than pretending the guarantee exists
- if M1 exposes only `strict` durability semantics for actual file creation, that limitation should be explicit and `normal` may remain an alias

For the create path in `strict` mode, M1 should also honor the M0 baseline durability contract:

- flush the newly initialized database file contents before reporting create success
- sync the parent directory entry where the current platform supports that guarantee
- fail create/open cleanly if those required durability steps fail

This milestone should optimize for honest guarantees over feature breadth.

Definition of done:

- the database handle cannot outlive its file resources
- unsupported writer-sharing scenarios are rejected clearly
- the project has a defined placeholder path from M1 guards to M7 locking

### 6.7 Define Error Classification At The Boundary

M1 should keep public errors aligned with the M0 API sketch. The code may use more detailed internal validation errors, but public lifecycle calls should report the following categories consistently:

| Condition | Public Error |
| --- | --- |
| path does not contain enough structure to be recognized as a database bootstrap | `FormatError` |
| recognizable `zi6db` bootstrap with bad magic, endianness, reserved fields, or checksum | `Corruption` |
| unsupported `format_major`, future `format_minor`, or unsupported required feature flag | `IncompatibleVersion` |
| metadata page size, version, slot id, or duplicated header field conflicts with the header or physical slot | `Corruption` |
| header checksum, metadata checksum, root checksum, or freelist checksum fails | `Corruption` |
| metadata points outside `page_count` or below `FIRST_DATA_PAGE_ID` | `Corruption` |
| selected metadata `page_count` exceeds the file's complete whole-page length | `Corruption` |
| metadata A/B generations are equal | `Corruption` |
| newer generation has lower `txn_id` than the older candidate | `Corruption` |
| required file read/write/sync/close operation fails | `IoError` |
| requested lock semantics cannot be guaranteed | `LockConflict` |
| lifecycle misuse such as double close or close with future active transactions | `InvalidState` |

Where `ChecksumMismatch` exists internally, M1 should either keep it private to validation helpers or document exactly where it is folded into `Corruption` at the public boundary.

### 6.8 Test Matrix

M1 should add automated tests for at least the following cases:

| Area | Case | Expected Result | M0 Source |
| --- | --- | --- | --- |
| lifecycle | create a new database and reopen it successfully | open succeeds and selects metadata A | `api-sketch.md`, `transaction-recovery.md` |
| lifecycle | reopen preserves selected metadata fields | `generation = 1`, `txn_id = 1`, `root_page_id = 3`, `freelist_page_id = 4`, `page_count = 5` | `storage-format.md` |
| header | open a file too small or unstructured to be a database bootstrap | `FormatError` | `api-sketch.md` |
| header | corrupt header magic in an otherwise recognizable fixture | `Corruption` | `storage-format.md`, `crash-and-corruption-test-spec.md` |
| header | invalid endianness marker | `Corruption` | `storage-format.md` |
| header | corrupt header checksum | `Corruption` | `storage-format.md` |
| header | unsupported major version | `IncompatibleVersion` | `storage-format.md` |
| header | future unsupported minor version | `IncompatibleVersion` | `storage-format.md` |
| header | unsupported required feature flag | `IncompatibleVersion` | `storage-format.md` |
| header | invalid page size below `4096` or above `65536` | `Corruption` | `format-constants.md` |
| header | required-zero reserved header field is non-zero | `Corruption` | `storage-format.md` |
| metadata | metadata A valid and metadata B invalid | select A | `transaction-recovery.md` |
| metadata | metadata A invalid and metadata B valid | select B | `transaction-recovery.md` |
| metadata | metadata A and B both valid with A newer | select A | `crash-and-corruption-test-spec.md` |
| metadata | metadata A and B both valid with B newer | select B | `crash-and-corruption-test-spec.md` |
| metadata | both metadata pages invalid | `Corruption` | `transaction-recovery.md` |
| metadata | metadata checksum corruption | candidate invalid; both invalid means `Corruption` | `crash-and-corruption-test-spec.md` |
| metadata | metadata page type mismatch | candidate invalid; both invalid means `Corruption` | `crash-and-corruption-test-spec.md` |
| metadata | metadata slot id does not match physical page | candidate invalid; both invalid means `Corruption` | `storage-format.md` |
| metadata | metadata page size differs from header page size | candidate invalid; both invalid means `Corruption` | `storage-format.md` |
| metadata | metadata format version or required feature fields differ from the validated header | candidate invalid; both invalid means `Corruption` | `storage-format.md` |
| metadata | equal-generation A/B state, identical or different | `Corruption` | `transaction-recovery.md` |
| metadata | newer generation with decreasing `txn_id` | `Corruption` | `transaction-recovery.md` |
| critical pages | selected root page id out of range | `Corruption` | `crash-and-corruption-test-spec.md` |
| critical pages | selected freelist page id out of range | `Corruption` | `crash-and-corruption-test-spec.md` |
| critical pages | selected root or freelist page id below `FIRST_DATA_PAGE_ID` | `Corruption` | `storage-format.md` |
| critical pages | root page type mismatch | `Corruption` | `crash-and-corruption-test-spec.md` |
| critical pages | freelist page type mismatch | `Corruption` | `crash-and-corruption-test-spec.md` |
| critical pages | root or freelist checksum mismatch | `Corruption` | `crash-and-corruption-test-spec.md` |
| I/O | truncated file before header prefix is complete | clear open failure, no panic | `crash-safety-matrix.md` |
| I/O | truncated file after valid header but before metadata/root/freelist pages | `Corruption` or `IoError` by documented boundary rule | `crash-safety-matrix.md` |
| I/O | selected metadata `page_count` exceeds complete pages in file | `Corruption` | `storage-format.md`, `crash-safety-matrix.md` |
| I/O | complete extra tail pages or partial tail bytes exist beyond selected `page_count` | open succeeds; tail is ignored and not added to freelist | `storage-format.md`, `crash-safety-matrix.md` |
| I/O | short read or short write in page helper | `IoError` | `crash-and-corruption-test-spec.md` |
| durability | strict create flush fails | `IoError`; incomplete file is not treated as valid | `page-lifecycle-and-commit-ordering.md` |
| durability | strict parent directory sync is unsupported by platform | documented fallback or `IoError`, but no silent stronger claim | `page-lifecycle-and-commit-ordering.md` |
| options | `read_only` open then future write attempt | `ReadOnlyTransaction` once write API exists | `api-sketch.md` |
| locking | unsupported `multi_process` writer guarantee | `LockConflict` or documented fail-fast error | `page-lifecycle-and-commit-ordering.md` |
| lifecycle | close releases guard state and file resources | later same-process reopen can proceed | `api-sketch.md` |
| lifecycle | close with future active transactions | `InvalidState` | `api-sketch.md` |

If practical during M1, add table-driven corruption tests so M4 can later extend the same harness for crash and recovery scenarios.

Definition of done:

- the test names describe user-visible behavior
- every M1 exit criterion maps to at least one automated test
- corruption cases fail with deliberate assertions instead of incidental runtime errors

## 7. Recommended Execution Order

To reduce rework, the M1 implementation should proceed in this order:

1. add format constants and public error definitions
2. implement the header codec and validation path
3. implement metadata, root, and freelist page codecs
4. implement page-offset and whole-page I/O helpers
5. implement `create()`
6. implement create-path flush behavior required for the initial `strict` durability contract
7. implement `open()` and metadata selection
8. implement `close()` and guard behavior
9. add corruption and reopen tests
10. document the exact build and test commands used in the PR

This order keeps the byte-level format work ahead of lifecycle code and keeps validation ahead of feature growth.

### 7.1 Dependency And Handoff Notes

M1 is a foundation milestone for agents working on later plans. It should make these handoffs explicit in code comments or implementation notes without implementing the later features:

- M2 depends on the empty root leaf page using the final leaf-page type and checksum shape.
- M2 depends on the format layer exposing leaf-page validation and offset helpers without forcing B+Tree mutation logic into `db.open()`.
- M3 depends on `DB` state keeping the selected metadata snapshot explicit and immutable until a later in-memory switch.
- M3 depends on the selected metadata slot being recorded so the first real commit can write the inactive slot and preserve strict A/B alternation.
- M4 depends on open-path metadata validation being deterministic and testable without special-case create logic.
- M4 depends on M1 tests distinguishing candidate invalidation from final open failure so crash fixtures can reuse the same validation path.
- M6 depends on the empty freelist page preserving the persisted `reusable` and `pending_free` sections, even when both are empty.
- M7 depends on M1 lock handling being conservative and honest about unsupported cross-process guarantees.

The M1 PR should call out any intentionally deferred behavior that might otherwise look like a missing safety property, especially around `normal` durability mode, `unsafe` mode, and `multi_process` locking.

M1 should also leave explicit non-implementation notes for later milestones:

- M2 owns `get`, `put`, `delete`, root splits, and leaf/branch structural mutation.
- M3 owns transaction objects, reader snapshots, writer exclusivity beyond the open/close guard, rollback, and in-memory metadata switching after commit.
- M4 owns fault-injection crash tests and commit-time dual-sync durability beyond the initial create/open baseline.
- M6 owns non-empty freelist reuse, overflow pages, and restart-time migration of eligible `pending_free` groups.

## 8. Acceptance Checklist

M1 should be considered complete only if all of the following are true:

- a new database file can be created from scratch
- that file can be reopened successfully without rewriting any header state
- `open()` reads the fixed bootstrap prefix before interpreting page-sized structures
- invalid header magic, checksum, or page-size data are rejected cleanly
- invalid header endianness, required-zero reserved fields, and duplicated header/metadata conflicts are rejected cleanly
- unsupported format versions are rejected as `IncompatibleVersion`
- unsupported required feature flags are rejected as `IncompatibleVersion`
- metadata A/B selection follows the M0 ordering rules
- metadata slot ids must match their physical A/B pages
- metadata selection rejects non-monotonic `txn_id` when `generation` ordering would otherwise pick a winner
- equal-generation metadata copies are rejected as `Corruption`
- the selected metadata's root and freelist pages are validated before open succeeds
- the selected metadata's `page_count` is backed by complete pages in the file
- unreachable complete tail pages or partial tail bytes are ignored, not treated as corruption or reusable free space
- fixed-size page I/O uses positional reads and writes only
- truncated-file and short-read paths return clear errors
- clearly unsafe early writer-sharing scenarios are blocked or fail fast
- create-path `strict` durability steps are either implemented or explicitly constrained to the supported platform behavior
- public error categories match the M0 API sketch and do not leak inconsistent internal validation names
- tests cover the success path, metadata selection, header validation, critical page validation, I/O failures, options, locking, and lifecycle behavior
- the implementation leaves the on-disk format compatible with M2-M4, without requiring redesign

## 9. Non-Goals

The following work should not expand the milestone:

- implementing the B+Tree insert/delete path
- exposing buckets, cursors, or the KV API
- implementing real commit ordering and sync-boundary tests beyond create/open baseline behavior
- adding freelist reuse or pending-to-reusable migration logic beyond empty-page initialization
- introducing `mmap` or alternative I/O paths
- shipping full cross-process locking semantics
- optimizing for performance before correctness and format validation are proven
- implementing crash-fault injection beyond deterministic create/open corruption and I/O-failure tests
- inferring free pages from unreachable tail pages
- repairing or rewriting a corrupted database during open
- changing M0 constants, page numbering, checksum coverage, or metadata selection rules

## 10. Exit Outcome

If M1 succeeds, `zi6db` will still be a very small database, but it will have the right kind of smallness: a validated file format, a trustworthy bootstrap path, and a storage skeleton that M2-M4 can extend without reworking the fundamental disk layout or open/recovery rules.
