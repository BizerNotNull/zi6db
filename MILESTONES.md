# zi6db Milestones

This document translates the goals in [ROADMAP.md](./ROADMAP.md) into a practical delivery plan.

The milestones are ordered to reduce implementation risk:

1. Freeze the storage and transaction design early.
2. Build a minimal but correct single-file engine first.
3. Add transactions and crash safety before expanding surface area.
4. Add developer-facing APIs and operational capabilities after the core is trustworthy.

## Release Cadence

- `v0.1.0-alpha`: M1-M2
- `v0.2.0-alpha`: M3
- `v0.2.1-alpha`: M4 crash-safe preview
- `v0.3.0-beta`: M5-M6
- `v1.0.0`: M7

## M0: Core Design Freeze

### Goal

Lock down the core storage and transaction model before implementation spreads across the codebase.

### Scope

- Choose page size and alignment rules.
- Define file header and metadata page layout.
- Define B+Tree node formats for internal and leaf pages.
- Define page allocation and page ownership model.
- Define dirty page lifecycle from mutation to commit or rollback.
- Define transaction commit model and write ordering.
- Define root page update strategy.
- Define metadata generation, transaction ID, and state selection rules.
- Define checksum coverage for critical structures.
- Define freelist transaction semantics and overflow page format reservation.
- Define fsync and write ordering rules for supported platforms.
- Define locking strategy for single-process and multi-process access.
- Define versioning and upgrade policy for the on-disk format.
- Define the top-level error model.

### Deliverables

- Storage format design note.
- Transaction and recovery design note.
- Page lifecycle and commit ordering design note.
- Public API sketch for open, transaction, bucket, and cursor types.

### Exit Criteria

- We can explain exactly how a commit updates disk state.
- We can explain exactly how startup recovery selects the valid database state.
- We can explain exactly how dirty pages, metadata, roots, and freelist state transition across commit and rollback.
- We can explain the required write and sync order for a crash-safe commit on Linux, macOS, and Windows.
- We can explain how format compatibility will be checked across versions.

## M1: Single-File Storage Skeleton

### Goal

Make `zi6db` reliably create, open, validate, and close a single database file.

### Scope

- Database file creation.
- Database file open and close.
- File header with magic bytes and version.
- Metadata pages with basic validation.
- Fixed-size page read and write helpers.
- Conservative single-process / multi-process misuse prevention for early builds.
- Cross-platform file handling baseline for Linux, macOS, and Windows.

### Deliverables

- `open/create/close` API.
- Initial file format implementation.
- Minimal lock or guard behavior that prevents unsafe multi-writer use.
- Tests for create, reopen, and invalid file detection.

### Exit Criteria

- A database file can be created and reopened.
- Invalid headers or unsupported versions fail with clear errors.
- Unsafe multi-process writer usage is blocked or fails fast with a clear error.
- Page I/O works reliably across supported platforms.

## M2: Minimal Ordered KV Engine

### Goal

Ship a minimal ordered key-value engine with correct persistence semantics, but without full transaction complexity yet.

### Scope

- Single logical key space.
- B+Tree page structure.
- Ordered raw byte keys and values.
- `get`, `put`, `delete`, `exists`.
- Key replacement and deletion semantics.
- Basic split logic for growing trees.
- Root split handling.
- Variable-length key/value encoding.
- Internal structural validation helpers for tree and page sanity.

M2 correctness is required, but it does not need a fully balanced delete path yet:

- Page merge or rebalance may be deferred unless required for correctness.
- Delete may leave underfull pages as long as lookup, persistence, and reopen behavior remain correct.

### Deliverables

- Minimal storage engine implementation.
- Correctness tests for inserts, updates, deletes, and reopen behavior.
- Internal checker coverage for page type, root sanity, and basic B+Tree consistency.
- Small benchmark harness for point lookup and insert workloads.

### Exit Criteria

- Keys are stored in lexicographic order.
- Point lookups are correct after reopen.
- Inserts, updates, and deletes pass persistence tests.
- Structural sanity checks catch malformed tree state during testing.

## M3: Transaction MVP

### Goal

Introduce real database transaction behavior with a small and predictable concurrency model, using the commit ownership and write-order rules frozen in M0.

### Scope

- Read-only transactions.
- Read-write transactions.
- Commit and rollback.
- Single writer semantics.
- Consistent reader snapshots.
- Transaction lifetime rules.
- In-memory and on-disk dirty page ownership consistent with the planned crash-safe commit path.

### Deliverables

- Transaction API.
- Snapshot visibility tests.
- Writer exclusivity tests.
- Rollback correctness tests.

### Exit Criteria

- Read transactions observe a stable snapshot.
- Only one write transaction can commit at a time.
- Rollback leaves no visible partial changes.
- The transaction implementation does not require a second semantic redesign when M4 durability is added.

## M4: Crash Safety And Recovery

### Goal

Make commits durable and crash-safe so the engine can recover to a valid state after interruption.

### Scope

- Crash-safe commit protocol.
- Dual metadata pages or equivalent recovery mechanism.
- Checksum or equivalent protection for critical metadata.
- Startup validation of headers and metadata.
- Detection of incomplete or invalid commits.
- Fault-injection coverage for metadata switch, page writes, and sync boundaries.
- Configurable sync policy.

### Deliverables

- Recovery implementation.
- Fault injection or crash simulation tests.
- Internal verify/checker path for metadata, page type, and reachable-page sanity.
- Durability policy documentation.

### Exit Criteria

- Simulated crashes do not leave the database in an unrecoverable state.
- Startup logic reliably selects the last valid committed state.
- Corrupted critical metadata is detected and reported.
- Recovery and checker tooling can distinguish invalid metadata, invalid page types, and unreachable critical pages.

## M5: Buckets And Cursor API

### Goal

Expand the engine into a developer-usable database API with namespacing and ordered iteration.

### Scope

- Bucket or table abstraction.
- Create, drop, and list buckets or tables.
- Cursor API with `first`, `last`, `seek`, `next`, and `prev`.
- Range scan support.
- Prefix scan support.
- Clear behavior for empty buckets and missing keys.

### Deliverables

- Bucket management API.
- Cursor and iterator API.
- Tests for forward and backward iteration.
- Tests for range and prefix scans.

### Exit Criteria

- Multiple isolated logical collections are supported.
- Iteration works without materializing full result sets.
- Range and prefix scans are correct and predictable.

## M6: Space Reuse And Large Values

### Goal

Handle long-lived database usage by supporting large values and reclaiming free space efficiently.

The on-disk format hooks for freelist and overflow pages must already be reserved by M0, even if full reuse behavior lands here.

### Scope

- Overflow page handling for large values.
- Free page tracking.
- Space reuse after deletes and updates.
- File size and page usage statistics.
- Free space reporting.
- Full implementation of the freelist or equivalent reuse policy that was format-reserved earlier.

### Deliverables

- Overflow page implementation.
- Free list or equivalent reuse mechanism.
- Database statistics API.
- Tests for space reuse behavior.

### Exit Criteria

- Large values can be stored and read back correctly.
- Deleted space can be reused by later writes.
- Statistics expose page usage and free space clearly.

## M7: Production Hardening

### Goal

Prepare `zi6db` for production-oriented use with integrity tooling, backup paths, platform coverage, and stronger validation.

### Scope

- File locking for multi-process misuse prevention.
- Read-only integrity check or verify API/tool.
- Consistent backup or snapshot export.
- Optional compaction or vacuum.
- Cross-platform CI coverage.
- Improved error messages.
- Expanded benchmark suite.
- Expanded tests for corruption, locking, and crash edge cases.

### Deliverables

- Verify API or command.
- Backup/snapshot capability.
- Optional compaction implementation.
- CI matrix for Linux, macOS, and Windows.
- Benchmark documentation and baseline numbers.

### Exit Criteria

- Invalid states and corruption cases are detected cleanly.
- Multi-process misuse is blocked or fails safely.
- Backup and restore flows are documented and tested.
- The project is ready for a `v1.0.0` stabilization pass.

## Priority Notes

The following items should be treated as non-negotiable quality bars, not optional polish:

- Crash safety.
- Stable and validated on-disk format.
- Clear transaction semantics.
- Cross-platform behavior.
- Strong tests for corruption and recovery.

The highest-risk dependency chain in the plan is M3, M4, and M6. They all depend on the same core decisions around page versions, commit ordering, metadata atomicity, and freelist behavior, so M0 must freeze those rules in enough detail to avoid redesign later.

The following items can stay behind feature flags or ship later within their milestone if necessary:

- Explicit sync policy tuning.
- Nested buckets.
- Compaction or vacuum.
- Extended benchmarks beyond core workloads.
