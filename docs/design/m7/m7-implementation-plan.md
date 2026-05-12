# M7 Implementation Plan: Production Hardening

This plan maps `M7: Production Hardening` from `MILESTONES.md` onto concrete implementation work. It assumes M1-M6 have already delivered the single-file engine, ordered KV operations, transactions, crash-safe recovery, buckets/cursors, overflow pages, freelist reuse, and database statistics.

M7 must not redesign the on-disk format or weaken the M0 commit, recovery, checksum, or freelist rules. The goal is to make the existing engine safer to operate, easier to diagnose, and ready for the `v1.0.0` stabilization pass.

M7 also depends on the explicit M6 handoff seams: reusable read-only walkers for
roots, buckets, freelist chains, and overflow chains; canonical live-page and
free-page classification; transaction-pinned statistics; and an internal
`oldest_active_reader_txn_id` provider that can be replaced by cross-process
reader state. If any of those seams are missing or incomplete, M7 must first
finish the seam without changing M6's on-disk semantics.

## 1. Goals

- Enforce cross-process misuse prevention through real file locking.
- Add a read-only verify API and CLI/tool path that reports corruption precisely.
- Add a consistent backup/snapshot export path.
- Optionally add compaction/vacuum as a controlled rewrite operation.
- Run supported tests on Linux, macOS, and Windows in CI.
- Improve public and diagnostic error messages.
- Expand benchmarks and capture baseline numbers.
- Expand corruption, locking, and crash-edge tests beyond the M0 baseline.
- Prove that M6 overflow, freelist, and pending-free behavior remains safe under
  cross-process readers and operational tooling.

## 2. M0 Constraints To Preserve

M7 must preserve these frozen rules:

- The database remains a single positional-I/O file.
- The header is immutable after creation.
- Metadata pages A and B are the only commit switch points.
- `generation` is the primary recovery ordering field.
- `txn_id` remains the visibility and freelist cutoff field.
- Visible root, freelist, branch, leaf, and overflow pages are never overwritten in place.
- `strict` commit success still requires non-metadata write and sync, metadata write and sync, and only then in-memory metadata switch.
- Startup recovery must validate header, metadata, selected root, and selected freelist before publishing state.
- Unreachable tail pages from failed commits are ignored, not silently reclaimed as repair.
- Pending-free pages become reusable only when no older reader can still see them.
- M7 tools must not turn ignored tail pages, unreachable failed-commit pages, or
  suspicious unreferenced pages into reusable space.
- Verify, backup, and compaction must use the same page validators and selected
  metadata rules as startup recovery, unless they explicitly report a deeper
  non-mutating finding.

## 3. Non-Goals

- No SQL layer, replication, network protocol, or client/server mode.
- No multi-writer concurrent commit model.
- No format rewrite unless a separate format-version milestone is approved.
- No best-effort repair command that mutates corrupted files in place.
- No weakening of `strict`, `normal`, or `unsafe` durability semantics.
- No `mmap` storage path unless it is explicitly proven to preserve M0 ordering and recovery semantics.
- No nested-bucket redesign unless already delivered before M7.

## 4. Deliverables

- File-locking implementation for supported platforms.
- Verify API and command/tool with structured diagnostics.
- Backup/snapshot API and command/tool.
- Optional compaction/vacuum API and command/tool behind an explicit feature gate or release decision.
- CI matrix covering Linux, macOS, and Windows.
- Error-message audit and improved contextual diagnostics.
- Benchmark suite documentation and baseline results.
- Expanded tests for corruption, locking, crash edges, backup/restore, and optional compaction.

## 5. Implementation Workstreams

### 5.1 File Locking

M7 should replace early conservative process-local guards with cross-process locks while keeping the same logical concurrency model: many readers, one writer.

Required behavior:

- Opening in writable mode obtains a writer-capable database-open lock that
  prevents incompatible writers from entering the process, or fails with
  `LockConflict`.
- Opening in `read_only` mode obtains a shared database-open lock when the
  platform supports it.
- A write transaction obtains or confirms exclusive writer ownership before it can commit.
- A read transaction records the selected `generation` and `txn_id` after the shared lock path confirms the database is safe to read.
- Lock acquisition must fail fast by default. Any wait/retry behavior must be explicit in options and bounded.
- Lock conflicts must never be reported as generic `IoError`.
- Locks must be released on `close()` only after all active transactions are closed.
- `close()` must continue to return `InvalidState` if transactions remain active.
- Locking must not rely on process-local state once `lock_mode = .multi_process`
  is enabled.
- Lock acquisition and release must be exception-safe around open, transaction
  begin, commit failure, rollback, and close failure paths.
- A failed `commit()` after metadata write begins must not release or advance
  reader-cutoff state in a way that assumes the new metadata won; reopen remains
  authoritative, matching M0 failure class B.

Platform strategy:

- Linux: use advisory byte-range locks through `fcntl`, with a documented lock byte/range layout.
- macOS: use the same `fcntl` advisory-lock strategy unless tests prove platform-specific behavior requires a wrapper.
- Windows: use `LockFileEx` / `UnlockFileEx` over the same logical lock ranges.
- Do not mix incompatible locking families for the same database path. For
  example, do not combine `flock` and `fcntl` for correctness-critical locks.
- Document that POSIX advisory locks only protect cooperating processes using
  the zi6db lock protocol. Non-cooperating writers remain outside the supported
  failure model from M0.
- Document Windows sharing flags used at file open. They must allow the intended
  shared readers and backup/verify operations while denying conflicting writers
  consistently with the byte-range locks.

Recommended lock ranges:

- Header/shared open range: protects compatible open and close lifecycle.
- Writer range: exclusive owner for write transactions and metadata publication.
- Reader-table range or sidecar-free equivalent: protects publication of active reader slots used for freelist reuse.

The lock-range layout must be treated as a runtime protocol, not a disk-format
field. It may reserve bytes in the file for OS locking, but it must not store
dynamic state in the immutable header page or require a format bump.

Reader tracking:

- M7 must provide a way to compute `oldest_reader_txn_id` across processes.
- The preferred design is an in-file or lock-protected reader table that does not change visible database format semantics.
- Stale reader detection must be conservative. A stale reader may delay reuse, but must not cause unsafe reuse.
- If a stale reader cannot be proven dead, its pinned `txn_id` remains active for reuse decisions.
- Reader slots must record enough identity to detect stale processes without
  trusting reused process IDs alone. Use at least process identity plus a slot
  generation, heartbeat, OS lock ownership, or another platform-proven liveness
  check.
- Reader-slot cleanup may only remove a slot after the implementation proves
  the owning process no longer holds the corresponding OS lock. Timeouts alone
  are not proof.
- `oldest_reader_txn_id` must include readers from all cooperating processes and
  must be sampled under the reader-table protocol before pending-free promotion.
- If reader-table initialization or validation fails, opening in
  `multi_process` mode must fail safely instead of falling back to process-local
  reuse decisions.

Tests:

- two writable opens conflict
- read-only open can coexist with a writer-capable open when allowed by the lock model
- second writer transaction fails with `WriteConflict` or `LockConflict` as appropriate
- process crash releases OS locks
- stale reader state does not allow unsafe freelist reuse
- Windows locking denies conflicting access with clear `LockConflict`
- POSIX tests prove cooperating processes using `fcntl` locks block conflicting
  writers and share compatible readers
- lock acquisition failure during `open()` cleans up the file handle and any
  partial reader-table state
- failed commit, rollback, and close release only the locks they are responsible
  for and never clear another process's reader slot

### 5.2 Verify API And Tool

Verification must be read-only. It must never repair, rewrite, truncate, compact, or migrate the database file.

Proposed API shape:

```zig
pub const VerifyOptions = struct {
    scan_pages: bool = true,
    scan_unreachable_pages: bool = false,
    check_freelist: bool = true,
    check_btree_order: bool = true,
    include_informational: bool = true,
    max_errors: u32 = 100,
};

pub const VerifyIssueKind = enum {
    invalid_header,
    incompatible_version,
    metadata_checksum_mismatch,
    metadata_conflict,
    invalid_page_range,
    invalid_page_type,
    page_checksum_mismatch,
    btree_order_violation,
    btree_child_range_violation,
    freelist_duplicate_page,
    freelist_reachable_page,
    overflow_chain_invalid,
    unreachable_page,
    ignored_tail_page,
    lock_conflict,
    io_failure,
};

pub fn verify(path: []const u8, options: VerifyOptions) !VerifyReport;
```

Finding model:

- `VerifyReport` must separate the API result from discovered findings. A clean
  database returns success with zero findings; a readable database with
  corruption findings returns a report rather than stopping at the first issue,
  up to `max_errors`.
- Fatal setup failures such as unreadable files, incompatible format,
  unsupported required feature flags, or lock acquisition failure may still
  return typed errors because no trustworthy report can be produced.
- Each finding must include severity (`error`, `warning`, or `info`), stable
  issue kind, optional page id, page type, metadata slot, generation, `txn_id`,
  logical owner, and a concise message.
- `error` findings indicate corruption or invariant violations that should make
  verification fail.
- `warning` findings indicate suspicious but non-authoritative data, such as
  optional unreachable-page scan results inside selected `page_count`.
- `info` findings include ignored tail pages outside selected `page_count` and
  other M0-allowed leftovers from failed commits.
- Tool exit code must be based on fatal errors and `error` findings, not on
  informational ignored-tail reports.
- JSON output must use stable field names and issue-kind strings before
  `v1.0.0`, so downstream tooling does not parse human prose.

Tool behavior:

- `zi6db verify <path>` exits `0` when no issues are found.
- It exits non-zero for corruption, incompatible format, lock conflict, or I/O failure.
- It prints a concise summary by default and detailed issue records with page id, page type, generation, and logical owner when available.
- It must support a machine-readable output mode, for example JSON, before `v1.0.0`.
- It must state whether unreachable-page scanning was enabled, because disabled
  deep scans are not proof that every physical page is meaningful.
- It must avoid dumping user key/value bytes by default. If a debug mode ever
  includes payload excerpts, it must be explicit and bounded.

Verification coverage:

- Header magic, version, endianness, page size, required feature flags, and header checksum.
- Metadata A/B validity, generation ordering, `txn_id` monotonicity, equal-generation rejection, and selected metadata decision.
- Root and freelist page ranges, page types, checksums, and generations.
- Reachability from bucket roots through branch, leaf, and overflow pages.
- B+Tree ordering, child key bounds, slot offsets, key/value payload ranges, and duplicate key constraints.
- Freelist reusable and `pending_free` duplicates, out-of-range pages, and overlap with reachable pages.
- Overflow chain length, ownership, duplicate pages, and broken links.
- Optional scan of unreachable pages to report suspicious but non-authoritative leftover data from failed commits.
- M6 handoff coverage for overflow references encoded through leaf value flags,
  freelist page chains, pending-free groups, reusable ranges, and transaction
  pinned statistics.

Traversal rules:

- Verification must begin by running the normal header and metadata selection
  algorithm. It must not invent a third recovery path.
- If both metadata slots are valid with equal generation, verify reports the M0
  equal-generation corruption instead of selecting either slot.
- If a metadata slot is invalid, verify may include it in diagnostic output, but
  reachability and ownership checks must be based on the selected valid slot.
- Reachability walkers must track ownership so a page cannot be simultaneously
  owned by a tree, an overflow chain, a freelist page, `reusable`, or
  `pending_free`.
- Unreachable physical pages below selected `page_count` are suspicious only
  when a full scan is requested; unreachable pages beyond selected `page_count`
  are ignored-tail informational findings.
- The verify API must reuse M6 validation helpers rather than reimplementing
  overflow or freelist decoding differently from normal reads.

### 5.3 Backup And Snapshot Export

Backup must produce a consistent database image without blocking readers for the full copy duration.

Required behavior:

- Backup starts from a pinned read transaction snapshot.
- The backup output must contain a valid header, exactly one selected metadata state, and all pages reachable from that snapshot.
- The backup must not include uncommitted pages from an active writer.
- The backup must not require exclusive writer shutdown, but it may briefly coordinate with writer metadata publication if needed.
- A restored backup must pass normal `open()` and `verify()`.
- Backup must preserve M6-visible state for the pinned snapshot, including
  bucket roots, overflow chains, freelist chains, `reusable` ranges, and
  `pending_free` groups.
- Backup must not promote pending pages, compact page IDs, infer free space, or
  include ignored tail pages. It is a snapshot copy, not a repair or vacuum.
- Backup must either copy pages to their original page IDs with the same
  `page_count`, or explicitly define a rewrite mode that updates every
  reference and is treated as compaction rather than backup.
- The required M7 backup mode is identity-preserving by page ID.

Proposed API shape:

```zig
pub const BackupOptions = struct {
    sync_output: bool = true,
    overwrite: bool = false,
};

pub fn backup(db: *DB, destination_path: []const u8, options: BackupOptions) !BackupStats;
pub fn backupToWriter(db: *DB, writer: anytype, options: BackupOptions) !BackupStats;
```

Snapshot algorithm:

1. Begin a read transaction and pin its metadata snapshot.
2. Enumerate all pages reachable from the selected root, freelist, buckets, and overflow chains.
3. Write a new destination file using positional writes.
4. Write the immutable header using the source format fields.
5. Copy all reachable pages needed by that snapshot to their original page IDs.
6. Zero-fill or otherwise create deterministic invalid bytes for non-copied
   pages below the pinned `page_count`, if the identity-preserving output keeps
   the same logical size.
7. Write one valid metadata slot for the pinned snapshot and leave the other
   metadata slot invalid.
8. Flush the destination file when `sync_output` is true.
9. Sync the parent directory for newly created destination files where supported.
10. Close the read transaction.
11. Reopen or verify the destination in tests before declaring success.

Metadata output rules:

- The output metadata must reference the same logical snapshot that the read
  transaction pinned.
- The output may keep the pinned metadata's `generation` and `txn_id`, but only
  one metadata slot may validate in the backup file.
- If the implementation rewrites metadata slot identity, it must still preserve
  M0 rules: no equal-generation valid slots, selected root and freelist in
  range, and header/metadata duplicated fields consistent.
- The backup must not copy the inactive source metadata slot as valid, because
  that could reintroduce an older or ambiguous source slot into the backup.

Failure rules:

- If output creation, write, or sync fails, the source database remains unchanged.
- Partial destination files should be removed when safely possible, or left with an explicit temporary suffix.
- Backup failure must report destination path and failed phase.
- `overwrite = false` must fail if the destination exists.
- `overwrite = true` must use a temporary file plus platform-appropriate rename
  rules, not destructive in-place overwrite of an existing backup.
- A failed backup must not hold a read transaction or reader slot after it
  returns.
- `backupToWriter` must document that it cannot provide parent-directory sync
  and may not be reopen-tested unless the writer is file-backed by the caller.

Tests:

- backup while writer is idle
- backup while writer commits after snapshot start
- restored backup sees the old pinned state, not a later commit
- backup of database with overflow pages
- backup of database with pending freelist groups
- destination `ENOSPC` or short write leaves source valid
- restored backup passes `verify()`
- backup excludes ignored tail pages from failed commits
- backup preserves pending-free groups without promoting them
- backup of a source with one invalid metadata slot produces exactly one valid
  destination metadata slot

### 5.4 Optional Compaction Or Vacuum

Compaction is optional for M7. If included, it must be treated as a rewrite/snapshot operation, not in-place page surgery.

Recommended release decision:

- Ship backup/snapshot as required.
- Ship compaction only if backup and verify are already stable.
- If schedule risk is high, document compaction as deferred and keep any API behind a feature flag.

Minimum preconditions before compaction may ship:

- `verify()` must be stable enough to validate the source snapshot and the
  compacted destination.
- Backup/snapshot export must already pass cross-platform tests.
- M6 live-page classification must distinguish tree pages, bucket roots,
  freelist pages, overflow chains, reusable pages, pending pages, and ignored
  tail pages without overlap.
- The rewrite code must have a complete page-id remapping layer for branch
  child pointers, bucket roots, leaf overflow references, freelist roots, and
  overflow next-page links.
- Source replacement must be a separate phase after destination-only compaction
  is correct and tested.
- If any of these preconditions are missing, M7 should defer compaction rather
  than ship a partial implementation.

Safe compaction model:

- Start from a read transaction snapshot.
- Create a new compacted file at a temporary path.
- Rewrite reachable pages densely into new page ids.
- Rebuild metadata, bucket roots, freelist, and overflow references for the compacted layout.
- Verify the compacted file.
- Atomically replace the destination only when the caller explicitly requests replacement and platform rename rules are satisfied.

API shape if implemented:

```zig
pub const CompactOptions = struct {
    sync_output: bool = true,
    replace_source: bool = false,
};

pub fn compact(db: *DB, destination_path: []const u8, options: CompactOptions) !CompactStats;
```

Rules:

- In-place compaction is not allowed.
- Source replacement requires exclusive database ownership.
- Replacement must sync both the compacted file and parent directory when supported.
- If replacement cannot be made crash-safe on a platform, M7 must expose destination-only compaction there.
- Compaction must start from a verified or verifier-compatible source snapshot.
  If the source snapshot contains corruption, compaction fails instead of
  attempting repair.
- The compacted output should normally contain no reusable or pending pages
  other than pages needed by its own empty freelist representation.
- The compacted output must not preserve ignored tail pages or unreachable
  failed-commit pages.
- The compacted output may reset physical layout, but it must preserve logical
  contents, bucket names, key order, transaction-visible value bytes, and format
  compatibility.
- Replacement of the source path is unsupported while any other process has the
  database open, even read-only.

Tests:

- compacted copy opens and verifies
- compacted copy preserves all keys, buckets, and overflow values
- compacted copy has no reachable page overlap with freed source pages
- crash before replacement leaves source valid
- crash during replacement follows documented platform behavior
- source with corruption is rejected without producing a trusted compacted file
- pending and reusable source pages are not copied as live pages
- page-id remapping updates branch, leaf, bucket, freelist, and overflow
  references consistently

### 5.5 Cross-Platform CI

M7 must establish CI as a release gate for supported platforms.

Required matrix:

- Linux latest stable image
- macOS latest stable image
- Windows latest stable image
- Debug and release-safe test builds where practical
- At least one job that runs benchmark smoke tests without enforcing strict timing thresholds

Realism rules:

- CI proves behavior on hosted Linux, macOS, and Windows runners, but it does
  not prove every filesystem, antivirus, network-drive, or power-loss behavior.
  Release notes must state the tested platforms and filesystem assumptions.
- Crash and fault-injection tests in CI should use deterministic injected
  failures in the storage layer. Real process-kill tests are useful but must not
  depend on timing sleeps.
- Power-loss semantics cannot be fully proven in hosted CI. M7 CI verifies the
  M0 write-ordering implementation and recovery model through fault injection.
- Any platform-specific skip must include a reason, owner, and issue link. A
  skipped mandatory correctness test blocks `v1.0.0` readiness unless the
  platform is explicitly downgraded from supported to experimental.
- Windows jobs must run path, locking, rename, and sharing-mode tests; POSIX
  jobs must run `fcntl` byte-range lock tests.

Required steps:

- `zig fmt --check` or equivalent formatting check
- `zig build`
- full unit test suite
- integration tests for create/open/transactions/recovery
- corruption and verify tests
- locking tests, including multi-process cases
- backup/restore tests
- optional compaction tests when enabled
- benchmark smoke run

CI expectations:

- Tests that require destructive crash/fault injection must use temporary directories.
- Flaky timing-based tests must be avoided. Use process barriers, lock files, or test harness IPC instead of sleeps.
- Platform-specific skips must include a reason and an issue link before `v1.0.0`.
- CI output should preserve failure artifacts for corrupted test files, verify reports, and crash logs.
- CI should upload benchmark smoke output as artifacts, but benchmark smoke
  should only fail on harness failure, correctness failure, or extreme invalid
  results such as zero successful operations.
- Long-running soak, stress, and large-dataset benchmarks may run on scheduled
  or manual workflows instead of every pull request, but their absence must be
  documented as residual release risk.

### 5.6 Error Messages And Diagnostics

M7 should keep public error types stable while adding contextual messages for humans and structured diagnostics for tools.

Scope boundary:

- Do not add a large new public error taxonomy unless a separate API review
  approves it. Prefer stable M0 error categories plus structured diagnostic
  context.
- Human messages may evolve, but verify JSON issue kinds and documented error
  categories should be stable before `v1.0.0`.
- Error messages must not expose user key/value payload bytes by default.
- Avoid promising exact OS error wording across platforms. Preserve raw OS error
  codes internally when useful, and map public categories consistently.

Required improvements:

- Attach operation context: `open`, `verify`, `commit`, `backup`, `compact`, `lock`, `sync`.
- Attach path context when safe: database path, destination path, temporary path.
- Attach structural context: page id, page type, metadata slot, generation, `txn_id`, expected value, actual value.
- Distinguish `FormatError`, `IncompatibleVersion`, `Corruption`, `LockConflict`, `WriteConflict`, `ReadOnlyTransaction`, `InvalidState`, and `IoError` in user-facing text.
- Report checksum failures as corruption at public boundaries, but preserve internal issue kind for verify reports.
- For ambiguous commit failures, explicitly state that reopen/recovery is authoritative.

Examples:

- `open failed: metadata slot B checksum mismatch at page 2; selected slot A remains valid`
- `verify failed: freelist page 42 contains page 107 also reachable from bucket "users"`
- `lock conflict: database already has an active writer lock`
- `backup failed while syncing destination file "backup.zi6db": no space left on device`
- `verify failed: overflow chain for page 81 references page 200 beyond selected page_count 150`

### 5.7 Benchmarks

Benchmarks must produce repeatable baseline numbers, not only ad hoc local timing.

Scope boundary:

- M7 benchmarks are for visibility and regression investigation, not hard
  performance guarantees.
- CI benchmark smoke tests must validate that workloads run and produce sane
  metrics. They should not enforce timing thresholds until a later performance
  policy defines stable hardware and variance handling.
- Baselines must be labeled by machine, OS, filesystem, Zig version, build mode,
  durability mode, page size, dataset size, and whether the database was cold or
  warmed.
- Benchmark datasets should avoid using production or sensitive data.

Required workloads:

- create and initial load
- sequential insert
- random insert
- point lookup
- range scan
- prefix scan
- update existing keys
- delete existing keys
- large value insert/read using overflow pages
- mixed read/write transaction workload
- backup throughput
- optional compaction throughput
- verify throughput on clean database

Required metrics:

- operations per second
- latency percentiles where the harness supports them
- database file size
- page count, free page count, overflow page count
- bytes written if measurable
- sync mode and platform
- Zig version and build mode

Baseline policy:

- M7 must document baseline numbers for a representative machine and CI smoke numbers.
- CI should not fail on benchmark regressions until a stable threshold policy exists.
- The benchmark document must state dataset sizes, key/value distributions, durability mode, and filesystem assumptions.
- Benchmark output should include the exact zi6db git commit and whether verify,
  backup, or compaction ran with deep scanning enabled.
- Any comparison against another database is optional and must be clearly marked
  as non-authoritative unless the harness is reviewed separately.

### 5.8 Expanded Tests

M7 tests should extend the M0 crash and corruption test spec instead of replacing it.

Test harness requirements:

- Multi-process tests must use explicit IPC barriers so the test knows when
  opens, locks, transactions, commits, and reader-slot writes have happened.
- Crash tests must inject failures at named storage phases rather than relying
  only on killing processes at arbitrary times.
- Every corruption fixture must state whether `open()` should fail immediately
  or whether only `verify()` with deep traversal should report the issue.
- Tests that mutate bytes must operate on temporary copies and must never reuse
  corrupted files as later test inputs unless the test says so explicitly.
- Locking and backup tests must run with paths containing spaces and non-ASCII
  characters where the platform supports them, because Windows and hosted CI
  path behavior often differs from POSIX defaults.

Corruption tests:

- header checksum mismatch
- metadata checksum mismatch in each slot
- equal-generation metadata conflict
- newer metadata with decreasing `txn_id`
- selected root out of range
- selected freelist out of range
- wrong page type for root, freelist, branch, leaf, and overflow pages
- page checksum mismatch in reachable branch, leaf, freelist, and overflow pages
- B+Tree child key range violation
- leaf slot payload offset out of bounds
- duplicate reachable page through two parents
- freelist page also reachable from root
- duplicate page in freelist reusable set
- duplicate page across `pending_free` groups
- overflow chain cycle
- overflow chain page shared by two live values
- ignored tail pages from failed commits are reported as info, not corruption
- unreachable page below selected `page_count` is reported only when the
  relevant verify scan option is enabled
- corrupt inactive metadata slot does not prevent opening when the active slot
  and selected critical pages are valid

Locking tests:

- writable open conflicts across processes
- shared read opens coexist when allowed
- writer transaction conflicts across processes
- process death releases OS locks
- stale reader table entry delays reuse until proven safe
- read-only verify can run while a writer-capable process is open
- backup can run while concurrent writer commits after backup snapshot begins
- reader slot from a killed process is cleaned only after OS lock/liveness proof
- unverifiable stale reader slot blocks pending-free promotion instead of
  allowing unsafe reuse
- lock conflict during `open()` does not leave a persistent reader-table slot
- read-only open cannot accidentally obtain writer capability
- writer-capable open cannot publish metadata after losing the writer lock

Crash-edge tests:

- crash before non-metadata page write
- crash during non-metadata page write
- crash after non-metadata sync before metadata write
- torn metadata write
- crash after metadata write before metadata sync
- crash after metadata sync
- `ENOSPC` during tail growth
- short write during metadata write
- sync failure before metadata write
- sync failure after metadata write with ambiguous outcome
- crash during backup destination write
- crash during optional compaction replacement
- crash after backup has copied pages but before destination metadata write
- crash after backup metadata write but before destination sync
- crash while writing reader-table state or after process death with active
  read transaction
- crash after pending-free promotion is computed but before the new freelist
  metadata is published
- crash during destination-only compaction before source replacement is requested

All crash tests must assert:

- whether `commit()` could have returned success
- which metadata slot should be selected or whether `Corruption` is required
- whether the final visible key/value state is old, new, or intentionally ambiguous until reopen
- whether verify reports the expected issue kind
- whether `oldest_reader_txn_id` and pending-free promotion remain conservative
  after reopen
- whether backup or compaction artifacts are either absent, invalid, or valid
  according to the documented phase boundary

Backup and snapshot tests:

- backup created from a long-running read transaction preserves that
  transaction's view across later commits
- backup does not copy pages outside selected `page_count`
- backup output has one valid metadata slot and one invalid slot
- restored backup preserves overflow values and bucket iteration order
- restored backup preserves freelist state enough for later writes to reuse
  pages safely
- backup to an existing path with `overwrite = false` fails before writing
- backup overwrite uses a temporary path and never corrupts an existing valid
  backup on failure

Verify API/tool tests:

- API returns structured findings with stable issue kinds and severities
- CLI summary and JSON mode agree on issue counts and exit status
- `max_errors` stops traversal deterministically and marks the report truncated
- lock conflict returns the documented category and does not masquerade as
  corruption
- machine-readable output omits user payload bytes by default

M6 handoff tests:

- verify walkers consume the same overflow and freelist validators used by M6
- cross-process reader tracking replaces the M6 process-local
  `oldest_active_reader_txn_id` provider without allocator changes
- backup and compaction use M6 live-page classification without duplicating
  page ownership rules

## 6. Execution Order

M7 should run in this order to reduce risk:

1. Audit the M6 handoff seams for walkers, freelist/overflow validators,
   transaction-pinned stats, and `oldest_active_reader_txn_id`.
2. Add the cross-platform lock abstraction and keep single-process
   compatibility tests passing.
3. Add deterministic multi-process lock tests and conservative cross-process
   reader tracking for freelist reuse.
4. Add verify traversal primitives using the existing recovery and M6 validators.
5. Define and freeze the verify finding model, then expose `verify()` API and
   `zi6db verify` human and JSON output.
6. Add backup/snapshot export on top of pinned read transactions, initially as
   identity-preserving page-ID backup.
7. Verify restored backups in tests on all supported platforms.
8. Improve error messages and structured diagnostics across open, commit,
   verify, lock, backup, and sync paths.
9. Add the CI matrix and make format, build, unit, integration, locking,
   verify, backup, and crash tests required.
10. Expand benchmark harness and publish baseline numbers without timing gates.
11. Decide whether compaction ships in M7 or is deferred.
12. If compaction ships, implement destination-only compaction first, then
   optional source replacement only after crash tests pass.
13. Run the full M7 acceptance checklist before declaring `v1.0.0`
   stabilization readiness.

## 7. Acceptance Checklist

- Cross-process multi-writer misuse is blocked or fails safely with `LockConflict`.
- Multiple supported read-only opens behave consistently with the chosen lock model.
- A writer cannot publish metadata unless it owns the writer path.
- Reader tracking provides a conservative `oldest_reader_txn_id` for freelist reuse.
- Stale reader state may delay page reuse but cannot make pending pages reusable
  early.
- Locking behavior is documented per platform, including advisory POSIX limits
  and Windows sharing-mode behavior.
- `verify()` is read-only and never mutates source files.
- `verify()` detects invalid header, metadata, root, freelist, B+Tree, overflow, and freelist states.
- Verify reports include severity, issue kind, page, slot, generation, `txn_id`,
  owner context, and truncation state when `max_errors` is reached.
- `zi6db verify <path>` has clear human output and a machine-readable mode.
- Verify distinguishes corruption from warnings and informational ignored-tail
  pages.
- Backup starts from a pinned snapshot and excludes later commits.
- Backup is identity-preserving by page ID unless a separate compaction path is
  explicitly invoked.
- Backup preserves overflow chains, freelist chains, `reusable`, and
  `pending_free` state for the pinned snapshot.
- Restored backups open normally and pass verify.
- Backup failure never changes the source database.
- Backup overwrite uses a temporary file and platform-appropriate rename/sync
  behavior.
- Optional compaction, if shipped, writes a verified destination file before any replacement.
- Optional source replacement, if shipped, is crash-safe or explicitly unsupported per platform.
- Optional compaction ships only after verify, backup, and M6 live-page
  classification are stable; otherwise it is documented as deferred.
- Linux, macOS, and Windows CI all run build, unit, integration, corruption, locking, backup, and crash-edge tests.
- CI uses deterministic fault injection and IPC barriers for crash/locking tests
  instead of flaky sleep-based timing.
- Error messages distinguish corruption, incompatible versions, lock conflicts, lifecycle misuse, and raw I/O failures.
- Error messages and diagnostics avoid exposing user payload bytes by default.
- Ambiguous commit failures tell callers that reopen/recovery is authoritative.
- Benchmarks cover point lookup, scans, writes, deletes, overflow values, verify, backup, and optional compaction.
- Baseline benchmark numbers and methodology are documented.
- Benchmark CI smoke is non-gating for timing regressions until a later threshold
  policy exists.
- All M0 crash matrix cases remain passing after M7 changes.
- M6 overflow, freelist, stats, and pending-free tests remain passing after
  cross-process reader tracking replaces process-local reader cutoff logic.
- No M7 change weakens the frozen M0 on-disk format, commit ordering, recovery selection, or freelist safety rules.
