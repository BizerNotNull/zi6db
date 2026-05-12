# M4 Feature Plan: Sync Policy Platform Mapping

## Goal

Define the M4 implementation plan for mapping `strict`, `normal`, and `unsafe` sync policies onto Linux, macOS, and Windows without changing the M0 durability contract.

This feature exists to keep sync behavior explicit at the storage boundary, so commit ordering, database creation, and crash testing all use one platform-aware policy layer instead of scattering OS-specific decisions through commit code.

## Scope

This feature plan covers:

- Linux, macOS, and Windows sync primitive mapping
- `strict` behavior for commit and initial database creation
- `normal` aliasing rules in the initial M4 implementation
- `unsafe` ordering guarantees
- directory sync behavior where the platform exposes a reliable path
- tests for policy dispatch, ordering, and failure handling
- acceptance criteria for M4

This feature plan does not define new on-disk structures, new durability labels, or any weakening of M0 commit ordering.

## Frozen Inputs

This plan must follow the constraints in:

- [docs/design/m4/m4-implementation-plan.md](/D:/代码D/zi6db/docs/design/m4/m4-implementation-plan.md)
- [docs/design/m0/page-lifecycle-and-commit-ordering.md](/D:/代码D/zi6db/docs/design/m0/page-lifecycle-and-commit-ordering.md)
- [AGENTS.md](/D:/代码D/zi6db/AGENTS.md)

The following M0 rules are non-negotiable for this feature:

- non-metadata pages are written before metadata
- `strict` requires a pre-metadata sync and a metadata sync
- metadata remains the only durable commit switch
- `unsafe` may skip flushes, but it may not expose partial in-process state
- new database creation in `strict` must flush file contents and sync the parent directory entry where the platform provides a reliable directory sync path

## Design Summary

M4 should implement sync policy selection through a small platform adapter with three responsibilities:

1. map the requested durability mode to concrete OS calls
2. apply the required sync boundaries during commit and file creation
3. document when a mode is an alias rather than a distinct implementation

The adapter should be used by both the database creation path and the commit state machine. The commit path should not contain ad hoc `if linux`, `if windows`, or `if strict` branches outside this adapter.

## Proposed Module Shape

The implementation should introduce a policy-focused internal module, such as `src/storage/sync_policy.zig` or an equivalent subsystem-aligned location. The exact path may change with the final source layout, but the separation of responsibilities should remain stable.

Suggested responsibilities:

- decode the user-facing durability enum
- expose `sync_data_boundary()`
- expose `sync_metadata_boundary()`
- expose `sync_new_file_contents()`
- expose `sync_parent_directory_entry()`
- expose capability flags for testing and diagnostics

Suggested inputs:

- file handle for the database file
- optional parent directory handle for strict create
- requested durability mode
- target operation kind: commit data boundary, commit metadata boundary, or create

Suggested outputs:

- success
- structured `IoError`
- test-visible record of which sync primitive was attempted

## Platform Mapping

### Linux

`strict` must use `fsync()` for both commit boundaries and for the initial file-content flush during strict database creation.

`normal` must alias `strict` in the initial M4 implementation. M0 allows a possible future `fdatasync()` mapping, but M4 should not adopt it until the repository has documented proof that file growth, metadata replacement, and crash recovery guarantees remain equivalent for this database format.

`unsafe` skips durable flushes, but it must still execute the same logical phase ordering:

- write and finalize non-metadata pages first
- write metadata second
- publish in-memory metadata last

For strict create on Linux:

- flush the database file with `fsync()`
- attempt to open the parent directory and `fsync()` it
- if the directory open or directory `fsync()` path is unavailable due to platform or environment limitations, surface a documented `IoError` rather than silently downgrading `strict`

### macOS

`strict` must use `fsync()` plus the conservative `fcntl(F_FULLFSYNC)` path where available. The implementation should treat `F_FULLFSYNC` as part of the strict promise rather than an optional optimization.

`normal` should initially alias `strict`. M0 leaves room for a later `fsync()`-without-`F_FULLFSYNC` path, but M4 should not split the behavior until the reduced path has a written safety argument and crash-test evidence.

`unsafe` skips durable flushes while preserving the same in-process ordering guarantees as every other platform.

For strict create on macOS:

- flush file contents with the strict file path
- sync the parent directory entry using the best reliable directory handle flow the implementation can support
- if directory sync is not available through the chosen Zig or OS abstraction, fail `strict` create rather than claiming success without the required guarantee

### Windows

`strict` must use `FlushFileBuffers()` for the database file at both commit boundaries and after new-file creation content is written.

`normal` must alias `strict` in M4.

`unsafe` skips `FlushFileBuffers()` calls while preserving data-before-metadata-before-publication ordering.

For strict create on Windows:

- flush the database file with `FlushFileBuffers()`
- attempt parent directory synchronization only if the implementation can obtain a supported directory handle and a reliable flush path
- if no reliable directory sync path exists in the chosen implementation layer, document Windows strict-create behavior explicitly in code comments and in M4 notes before shipping the feature

The key rule is that M4 must not claim a stronger strict-create guarantee on Windows than the implementation can actually enforce.

## Normal Aliasing Rules

M4 should treat `normal` as an explicit alias of `strict` on all supported platforms.

Implementation requirements:

- the mode remains visible at the API boundary
- policy dispatch records that `normal` resolved to the strict backend
- tests assert aliasing directly instead of assuming it indirectly
- comments explain that the alias is temporary and deliberate

This keeps the API stable without introducing platform-specific under-validated behavior in M4.

## Strict Create Sync

Strict database creation must be modeled as a separate policy flow rather than as a partial reuse of commit logic.

Required create ordering:

1. create and size the new database file
2. write header, metadata slots, and any fixed bootstrap pages
3. flush file contents with the platform strict primitive
4. sync the parent directory entry where the platform exposes a reliable path
5. return success only after both required steps complete

Failure rules:

- any create-path sync failure returns `IoError`
- the caller must treat the create attempt as failed, even if some bytes reached disk
- M4 does not need to salvage a partially created file in place

Implementation notes:

- directory sync should be attempted only after file-content flush succeeds
- the parent directory handle should be acquired from the final database path, not from process cwd assumptions
- tests should distinguish file flush failure from directory sync failure

## Directory Sync Strategy

Directory sync is only relevant when the operation creates or replaces a directory entry whose visibility must survive power loss.

M4 needs directory sync for:

- initial database creation in `strict`

M4 does not need directory sync for:

- ordinary in-place commit metadata slot rotation inside an existing file
- `normal` while it aliases `strict`, except where the strict create path is reused
- `unsafe`

Implementation rules:

- keep directory sync logic out of the ordinary commit path
- make directory sync best-effort only when the platform truly lacks a reliable mechanism and the repository has explicitly documented that limitation
- otherwise, missing required directory sync support is a blocker for claiming strict-create success

## Unsafe Ordering Guarantees

`unsafe` is allowed to weaken persistence, but not logical visibility.

The implementation must preserve these guarantees:

- all referenced data pages are fully finalized before metadata is built
- all referenced data pages are written before metadata write begins
- metadata is written before the in-memory current metadata pointer switches
- `commit()` success is returned only after in-memory publication completes
- any failure before publication leaves the old in-memory metadata active

`unsafe` must not:

- publish metadata in memory before metadata bytes are written
- reorder metadata write ahead of data-page writes
- skip checksum finalization
- weaken failure terminal-state behavior

This means `unsafe` changes flush behavior only. It does not change the commit phase graph.

## Commit Integration

The commit state machine should call the sync policy adapter at exactly these boundaries:

1. after all non-metadata page writes complete
2. after inactive metadata slot write completes

Expected behavior by mode:

- `strict`: both calls execute the platform strict primitive
- `normal`: both calls dispatch through the normal label but resolve to the strict backend
- `unsafe`: both calls become no-op success results, while phase recording still shows the boundary was reached

The phase recorder used by fault injection should capture:

- requested durability mode
- resolved backend behavior
- whether a real OS sync call was attempted
- whether directory sync was attempted on strict create

## Failure Handling

The sync policy layer must preserve M4 failure-class behavior:

- failure before metadata write stays Class A
- failure after metadata write begins but before metadata sync is confirmed stays Class B
- success after the required metadata boundary stays Class C

Specific rules:

- a pre-metadata sync failure returns `IoError` and leaves in-memory metadata unchanged
- a metadata sync failure returns `IoError` and leaves in-memory metadata unchanged in the current process
- an `unsafe` no-op boundary must not introduce a new failure point
- strict-create directory sync failure returns `IoError` even if file bytes were already flushed

## Test Plan

### Unit Tests For Policy Mapping

Add table-driven tests for policy resolution:

- Linux `strict` resolves to `fsync()`
- Linux `normal` resolves to strict alias
- Linux `unsafe` resolves to no-op
- macOS `strict` resolves to `fsync()` plus `F_FULLFSYNC`
- macOS `normal` resolves to strict alias
- macOS `unsafe` resolves to no-op
- Windows `strict` resolves to `FlushFileBuffers()`
- Windows `normal` resolves to strict alias
- Windows `unsafe` resolves to no-op

These tests should validate both the selected backend and the documented capability flags exposed to diagnostics.

### Commit Ordering Tests

Add tests around the commit phase recorder:

- `strict` executes data sync before metadata write publication
- `normal` follows the same order as `strict`
- `unsafe` records both boundaries as skipped but still reaches them in order
- no mode publishes in-memory metadata before the metadata boundary handler returns success

### Strict Create Tests

Add create-path tests for:

- file flush attempted before directory sync
- directory sync attempted only for `strict`
- `normal` create follows the same strict-create path while aliasing remains in effect
- `unsafe` skips both flushes
- directory sync failure returns `IoError`
- file flush failure prevents directory sync attempt

### Fault Injection Tests

Add policy-aware fault injection coverage for:

- Linux strict pre-metadata `fsync()` failure
- Linux strict metadata `fsync()` failure
- macOS strict `F_FULLFSYNC` failure after `fsync()` succeeds
- Windows strict `FlushFileBuffers()` failure at both boundaries
- strict-create file flush failure
- strict-create directory sync failure

Each test should assert:

- commit or create return value
- whether in-memory metadata changed
- whether reopen may select old or new state when the failure is Class B
- which sync backend was attempted

### Cross-Mode Behavior Tests

Add mode-comparison tests asserting:

- `normal` and `strict` have identical observed commit behavior in M4
- `unsafe` skips durable flushes but not logical ordering
- `unsafe` success still requires metadata write completion before publication

## Acceptance

- Linux `strict` is implemented with `fsync()` at both commit boundaries.
- macOS `strict` is implemented with `fsync()` and the conservative `F_FULLFSYNC` path where available.
- Windows `strict` is implemented with `FlushFileBuffers()`.
- `normal` is an explicit alias of `strict` on Linux, macOS, and Windows in M4.
- `unsafe` skips durable flushes without changing commit phase ordering.
- commit code calls the sync policy layer only at the two M0 sync boundaries.
- strict create flushes file contents before attempting parent directory sync.
- strict create attempts parent directory sync only where the platform exposes a reliable path.
- the implementation does not silently downgrade `strict` to weaker durability.
- sync failures before publication leave in-memory metadata unchanged.
- tests cover platform mapping, aliasing, strict create ordering, directory sync failures, and unsafe ordering guarantees.

