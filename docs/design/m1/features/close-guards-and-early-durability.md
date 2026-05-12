# M1 Feature Plan: Close Guards And Early Durability

This plan narrows the M1 lifecycle work from
`docs/design/m1/m1-implementation-plan.md` into concrete implementation tasks for
close behavior, conservative open guards, and the first durability guarantees. It
must preserve the M0 API sketch and the page lifecycle rules without promising
cross-process locking or full transaction behavior before later milestones.

## 1. Scope

M1 should implement the smallest honest lifecycle surface:

- `DB.create`, `DB.open`, and `DB.close` maintain explicit handle state.
- `lock_mode = .single_process` prevents unsafe same-process writable opens.
- `lock_mode = .multi_process` fails fast until M7 implements real file locks.
- `read_only` opens may validate and read the file but cannot start write paths.
- `strict` create flushes initialized file contents before reporting success.
- `normal` is reserved and may be implemented as an alias of `strict`.
- `unsafe` is reserved and must not imply power-loss durability.

This feature must not implement transaction commit, B+Tree mutation, freelist
reuse, crash-fault injection, or production cross-process locking.

## 2. Close Lifecycle

The `DB` object should have a small explicit state machine:

| State | Meaning | Allowed Next States |
| --- | --- | --- |
| `opening` | file, format, and guard setup are in progress | `open`, `closed` on failure cleanup |
| `open` | handle is usable and guard state is held | `closing` |
| `closing` | `close()` is releasing resources | `closed` |
| `closed` | resources and guards are released | none |

Implementation rules:

- `close()` returns `InvalidState` for a handle that is already `closed` or
  currently `closing`.
- `close()` returns `InvalidState` if active transactions exist. M1 has no real
  transactions, so keep counters initialized to zero and wire the check now for
  M3.
- `close()` releases the process-local guard before returning success.
- `close()` releases the file handle even if future non-critical cleanup paths
  fail, but required close or sync errors still map to `IoError`.
- Failed `open()` or `create()` must clean up any partially acquired guard and
  file resources before returning.
- No public method may operate on a `closed` handle; lifecycle misuse maps to
  `InvalidState`.

The in-memory `DB` state should record at least:

- canonical or normalized path key used by the single-process guard
- whether the handle is `read_only`
- selected `lock_mode`
- selected `durability_mode`
- file handle ownership
- selected metadata snapshot and selected metadata slot
- active read transaction count, initialized to zero
- active write transaction flag, initialized to false
- lifecycle state

## 3. `single_process` Guard

M1 should guarantee conservative same-process safety before transaction locking
exists. The guard lives outside the file format layer, preferably in
`src/storage/lock.zig` or an equivalent storage lifecycle module.

Rules:

- Guard keys are based on a stable absolute path where practical. If full
  canonicalization fails because the path does not exist yet, create/open should
  use the best available absolute parent plus file name and keep the behavior
  documented.
- A writable `single_process` open conflicts with any existing same-path handle.
- A `read_only` `single_process` open may coexist with other `read_only`
  handles for the same path.
- A `read_only` `single_process` open conflicts with an existing writable handle
  until M3/M7 define broader reader/writer sharing.
- Guard acquisition happens before publishing the `DB` handle.
- Guard release happens exactly once during successful close or failed setup
  cleanup.
- Conflicts return `LockConflict`, not `IoError`.

This is intentionally stricter than the eventual MVCC model. M3 may relax
same-process read sharing around a writer once transaction snapshots exist.

## 4. `multi_process` Fail-Fast

M0 allows cross-process write access to be rejected before M7. M1 must not
pretend that `lock_mode = .multi_process` is safe.

Rules:

- `create()` or writable `open()` with `lock_mode = .multi_process` returns
  `LockConflict`.
- `read_only` plus `multi_process` may also return `LockConflict` in M1 unless
  the implementation can prove it does not weaken future guarantees.
- The error should occur before format mutation or create-path initialization.
- The public documentation and tests should call this an intentionally deferred
  M7 capability, not a platform limitation.

M7 may replace this fail-fast branch with shared/exclusive file locks, but it
must keep the same public safety level or stronger.

## 5. `read_only` Guard

`read_only` is a database-handle property, not just a file-open flag.

Rules:

- `open(read_only = true)` opens the file without write permissions where the
  platform supports that distinction.
- `create(read_only = true)` returns `InvalidState` because creating a new file
  is a mutating operation.
- Future `beginWrite`, bucket creation, delete, put, freelist mutation, and
  metadata publication paths must check `db.read_only` before touching storage.
- The future write entrypoint should return `ReadOnlyTransaction`.
- M1 page validation may read header, metadata, root, and freelist pages through
  the same positional read helpers used by writable opens.

M1 should add the guard field and tests now even if no mutating KV API exists.

## 6. Strict Create Flush

M1 has no commit path yet, but a successful `create()` must not report a
database as created before the initialized file state has been durably pushed as
far as the selected mode promises.

For `durability_mode = .strict`:

1. Create the file with exclusive creation semantics.
2. Write page `0` header, metadata A on page `1`, invalid metadata B on page
   `2`, empty root page on page `3`, and empty freelist page on page `4`.
3. Flush the database file contents with the strict platform primitive from
   `page-lifecycle-and-commit-ordering.md`.
4. Sync the parent directory entry where the platform supports that guarantee.
5. Only then publish the `DB` handle and return success.

Failure rules:

- Any required write, flush, or directory sync failure returns `IoError`.
- If strict directory sync is unsupported on a platform, the implementation must
  either return `IoError` or document a weaker platform-specific fallback in the
  M1 PR. It must not silently claim a stronger guarantee.
- Failed create cleanup may leave an incomplete file on disk, but a later
  `open()` must reject it through normal header, metadata, or critical-page
  validation.
- Create must not produce two valid metadata slots with the same generation.

## 7. `normal` And `unsafe` Reservations

M1 should expose only behavior it can honestly provide.

| Mode | M1 Behavior | Public Claim |
| --- | --- | --- |
| `strict` | file flush plus supported parent directory sync during create | initialized database is durable after successful create within platform model |
| `normal` | allowed to call the same implementation as `strict` | reserved lighter mode; no weaker behavior in M1 |
| `unsafe` | may be rejected with `InvalidState` or implemented without durable flushes | no power-loss durability promise |

Implementation notes:

- If `normal` aliases `strict`, keep one shared code path and a test that proves
  it still creates a reopenable file.
- If `unsafe` is accepted, tests must avoid asserting power-loss durability.
- `unsafe` must still avoid publishing an in-process handle until the initialized
  pages are fully written and internally self-consistent.
- Durability mode selection should be stored on `DB` so M3 can reuse it for
  commit sync boundaries.

## 8. Implementation Tasks

Recommended order:

1. Add lifecycle state and option fields to the `DB` skeleton.
2. Add a process-local path registry for `single_process` guard acquisition and
   release.
3. Add early rejection for unsupported `multi_process` opens.
4. Add `read_only` validation for create and future write-entry guards.
5. Add strict file flush and parent directory sync wrappers in the storage layer.
6. Wire create/open failure cleanup so guards and files are not leaked.
7. Wire `close()` state transitions, active transaction checks, and idempotent
   internal cleanup.
8. Add tests for lifecycle, guard, read-only, durability mode, and cleanup
   behavior.

The format layer should not know about guard state, lifecycle state, or
durability mode names except through storage sync calls made by `db.create()`.

## 9. Test Matrix

| Area | Case | Expected Result |
| --- | --- | --- |
| close lifecycle | close a newly created database | success; file and guard are released |
| close lifecycle | close a reopened database | success; later same-path open can proceed |
| close lifecycle | close twice | `InvalidState` on second close |
| close lifecycle | close with active read count | `InvalidState`; handle remains open |
| close lifecycle | close with active write flag | `InvalidState`; handle remains open |
| setup cleanup | create fails after guard acquisition | guard is released; later create/open is not blocked by leaked state |
| setup cleanup | open fails during header or metadata validation | guard and file resources are released |
| single_process | writable open while same path writable handle is open | `LockConflict` |
| single_process | writable create while same path read-only handle is open | `LockConflict` |
| single_process | read-only open while same path writable handle is open | `LockConflict` in M1 |
| single_process | multiple read-only opens for same path | success if implemented; otherwise documented `LockConflict` |
| multi_process | writable open with `lock_mode = .multi_process` | `LockConflict` |
| multi_process | create with `lock_mode = .multi_process` | `LockConflict` and no format mutation |
| read_only | create with `read_only = true` | `InvalidState` |
| read_only | open with `read_only = true` | validates selected metadata without writes |
| read_only | future write entrypoint on read-only DB | `ReadOnlyTransaction` |
| strict durability | successful strict create | file contents flushed before success; reopen succeeds |
| strict durability | file flush failure | `IoError`; no published handle |
| strict durability | parent directory sync failure where required | `IoError` or documented fallback; no stronger claim |
| normal reservation | create with `normal` when aliased to `strict` | success and reopenable file |
| unsafe reservation | create with `unsafe` if accepted | no power-loss durability assertion; still no partially initialized handle |

Tests should use behavior-oriented names and should not depend on incidental OS
error strings. Where flush or close failures are hard to force with real files,
introduce a small storage test double at the storage boundary rather than
special-casing format code.

## 10. M7 Handoff

M1 must leave a clean seam for M7 locking:

- Keep guard acquisition behind a small lock module interface.
- Store the requested `lock_mode` and actual acquired guard kind separately.
- Keep `multi_process` rejection centralized so M7 can replace it with file lock
  acquisition.
- Do not encode lock ownership into the database file.
- Do not rely on process-local guards for recovery correctness.
- Document that M1 same-process rules are conservative placeholders, not the
  final reader/writer concurrency model.

M7 should be able to add shared read locks, exclusive writer locks, stale-lock
handling if needed by the platform, and cross-process conflict tests without
changing the M0 disk format or M1 open-path validation order.

## 11. Acceptance Checklist

This feature is complete when:

- `close()` releases file resources and process-local guard state.
- `close()` rejects double-close and active-transaction cases with
  `InvalidState`.
- Failed create/open setup does not leak guard state.
- `single_process` blocks clearly unsafe same-process writer sharing.
- `multi_process` fails fast instead of making unimplemented safety promises.
- `read_only` is stored on `DB` and blocks current or future write-oriented
  entrypoints.
- `create(read_only = true)` is rejected.
- `strict` create flushes initialized file contents before returning success.
- Parent directory sync behavior is implemented or explicitly constrained by
  platform notes.
- `normal` and `unsafe` behavior is reserved without overstating durability.
- Public errors use `LockConflict`, `InvalidState`, `ReadOnlyTransaction`, and
  `IoError` consistently with `api-sketch.md`.
- Tests cover close lifecycle, guards, read-only behavior, strict create flush,
  mode reservations, and cleanup after setup failure.
- The implementation notes identify this as an M1 placeholder path that M7 will
  replace with production cross-process locking.
