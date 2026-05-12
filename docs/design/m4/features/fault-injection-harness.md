# M4 Feature Plan: Fault Injection Harness

This plan defines the concrete M4 implementation work for the crash-safety
fault injection harness. It refines
`docs/design/m4/m4-implementation-plan.md` using the reopen outcomes frozen in
`docs/design/m0/crash-safety-matrix.md`.

The goal is to make crash-path testing deterministic, readable, and cheap
enough to run continuously while still proving the M0 rule that metadata is the
only commit switch point.

## 1. Scope

This feature covers:

- a test-only storage adapter that can inject write, sync, and crash faults
- stable commit phase names shared by commit code and crash tests
- deterministic short-write and torn-write simulation for metadata and normal
  pages
- sync uncertainty simulation for the Class B boundary
- reopen-based assertions after injected crashes and ambiguous failures
- deterministic fixture builders for single-commit and multi-generation cases
- CI-safe execution limits and test partitioning

This feature does not cover:

- changing the on-disk format, metadata layout, or checksum rules
- repairing corrupted files or salvaging partial commits
- randomized fuzzing as the primary M4 validation strategy
- public user-facing failpoint controls
- performance benchmarking beyond basic CI budget constraints

## 2. Inputs And Frozen Constraints

The harness must preserve these already-frozen rules:

- metadata is the only durable commit switch point
- startup recovery through normal `open()` is the only authority after an
  ambiguous metadata write or sync result
- equal-generation metadata slots are always `Corruption`
- unreachable pages left behind by failed commits are not corruption by
  themselves
- in-process readers must stay on old metadata when `commit()` returns `IoError`
  before publication
- fault injection must be unavailable in production builds

## 3. Harness Shape

M4 should add a file adapter layer beneath the storage engine and above raw OS
file calls. The adapter should be selected only in tests or in a dedicated
test-only build mode.

Recommended shape:

```zig
const FaultHarness = struct {
    script: FaultScript,
    recorder: FaultRecorder,
    inner: FileLike,
};
```

Responsibilities:

- intercept tail growth, positional writes, and sync calls
- record the current commit phase and the last completed durable boundary
- apply deterministic mutations or failures exactly once when configured
- allow clean passthrough behavior when no fault is armed
- expose inspection state to tests without bypassing normal recovery

The database should not know whether the underlying file is fault-injecting
beyond a narrow storage interface. M4 tests should prove behavior through the
same commit and open entry points used by production code.

## 4. Stable Phase Names

The harness and the commit state machine must share one stable phase enum.

| Phase | Meaning | Typical failpoints |
| --- | --- | --- |
| `reserve_tail_pages` | reserve or grow the file for new page ids | file growth failure, crash after tail extension |
| `write_data_pages` | write branch, leaf, overflow-if-any, and freelist pages | short data write, torn page, crash after some writes |
| `sync_data_pages` | execute the pre-metadata sync boundary | sync failure before confirmation, crash after successful data sync |
| `build_metadata` | finalize inactive-slot metadata bytes and checksum in memory | crash before any metadata byte is written |
| `write_metadata` | begin and complete the inactive metadata slot write | short metadata write, torn metadata write, crash after full write |
| `sync_metadata` | execute the post-metadata sync boundary | uncertain sync result, crash after confirmed sync |
| `publish_metadata` | switch the in-memory selected metadata snapshot | assertion-only checkpoint; no fallible I/O allowed |

Rules:

- tests must be able to stop before and after every phase
- the recorder must expose both `current_phase` and `last_completed_phase`
- no persistence failure may be injected after `publish_metadata`
- `build_metadata` is included so tests can distinguish pre-write crashes from
  metadata-write ambiguity

## 5. Fault Model

The harness should support a compact set of deterministic fault actions rather
than an open-ended callback API.

### 5.1 Write Faults

Supported write faults:

- fail before writing any byte
- short write at byte offset `0`
- short write at a middle offset
- short write at `page_size - 1`
- write mutated bytes and then report failure
- write full bytes and then simulate immediate process crash before the next
  phase

Write fault targeting must support:

- tail growth or allocation writes
- any normal page write in `write_data_pages`
- specifically the freelist page write
- the inactive metadata slot write
- deterministic page ordinal targeting such as "first data page write" or
  "metadata write"

### 5.2 Crash Faults

Crash simulation means the test process stops the commit path immediately,
retains whatever bytes were already written by the adapter, and then reopens the
database from disk through normal `open()`.

Required crash points:

- before any data-page write
- after one or more data-page writes
- after all data-page writes but before `sync_data_pages`
- after a successful data sync but before metadata write
- during metadata write after partial bytes
- after a full metadata write but before metadata sync
- after a successful metadata sync and before any later test assertion

### 5.3 Sync Uncertainty

The harness must distinguish sync failures that are definitely pre-durable from
failures whose durability result is uncertain.

Required sync outcomes:

- `fail_before_issue`: no flush was issued; state is unchanged relative to the
  previous phase
- `fail_after_uncertain_issue`: the OS call was made but persistence cannot be
  confirmed; recovery decides the winner on reopen
- `success`: the configured sync boundary completed

`sync_data_pages` uncertainty is still Class A because metadata is unwritten.
`sync_metadata` uncertainty is Class B because the inactive metadata slot may
already be durable and valid.

## 6. Reopen Assertions

Every crash or ambiguous-failure test must assert both the in-process result and
the post-reopen result.

### 6.1 In-Process Assertions

Before reopening, tests must assert:

- whether `commit()` returned success, `IoError`, or a simulated crash outcome
- the write transaction ended in a terminal state
- the current process still exposes the old metadata snapshot for every Class A
  and Class B result
- `publish_metadata` was not observed on any failure path

### 6.2 Reopen Assertions

After reopening through normal `open()`, tests must assert:

- the selected metadata slot, when slot behavior is relevant
- the selected `generation`
- the selected `txn_id` when both slots validate
- whether open succeeded, returned `Corruption`, or returned
  `IncompatibleVersion`
- whether the recovered key/value view matches the old commit or the new commit

### 6.3 Outcome Table

| Fault class | In-process expectation | Reopen expectation |
| --- | --- | --- |
| Before metadata write begins | `commit()` fails or crashes; old metadata remains selected | old generation must win |
| Metadata write partially applied | `commit()` fails or crashes; old metadata remains selected | newest valid slot wins; torn metadata is discarded |
| Metadata fully written, metadata sync uncertain | `commit()` returns `IoError`; old metadata remains selected | old or new generation may win, but only by normal validation and generation rules |
| Metadata sync confirmed | `commit()` returns success only after publication | new generation must win |

M4 tests must not treat a returned `IoError` after `write_metadata` begins as
proof that the new commit is absent on disk.

## 7. Deterministic Fixtures

The harness needs fixtures that keep failures reproducible and assertions easy
to read.

### 7.1 Fixture Families

Required fixture families:

- `empty_db_fixture`: freshly created database with the initial committed state
- `single_leaf_fixture`: one committed leaf-root tree with a small stable key
  set
- `split_tree_fixture`: committed multi-page tree with at least one branch and
  multiple leaves
- `freelist_activity_fixture`: committed state with pending-free and reusable
  freelist content
- `dual_generation_fixture`: old and new committed generations used to prove
  slot alternation and reopen selection

### 7.2 Determinism Rules

Fixture builders should:

- use fixed page sizes and fixed insertion order
- use stable key/value bytes rather than random payloads
- commit a known number of transactions before the injected one
- record the expected old and new metadata tuples: slot, generation, `txn_id`,
  root page id, and freelist page id
- avoid dependence on wall-clock time, scheduler timing, or filesystem ordering
  outside the explicit harness script

### 7.3 Mutation Fixtures

The harness must also support deterministic post-commit byte mutation for:

- metadata page checksum corruption
- metadata slot id corruption
- root page checksum corruption
- freelist page checksum corruption
- arbitrary normal-page byte flips used by checker assertions

These mutations should be scripted by page role and byte offset, not by fragile
absolute file offsets alone.

## 8. CI Constraints

The harness must be strong enough to catch regressions without turning M4 tests
into a timeout risk.

CI constraints:

- keep the default suite deterministic and bounded; no unseeded randomness
- prefer table-driven scripts over combinatorial explosion
- run a small representative matrix on every PR
- keep expensive cross-product coverage in a separate slower test group if
  needed
- avoid per-test fixture rebuilds when one read-only base image can be copied
- limit each failpoint scenario to one injected event unless a test explicitly
  proves multi-event behavior

Recommended PR-tier coverage:

- one passing commit case per fixture family
- one Class A case before metadata write
- one torn normal-page write case
- one torn metadata write case
- one metadata sync uncertainty case
- one successful reopen-to-new-generation case
- one corruption mutation case each for metadata, root, and freelist

If runtime grows beyond the CI budget, reduce fixture count before reducing the
required Class A, Class B, and Class C coverage.

## 9. Non-Goals

This feature intentionally does not try to solve:

- generalized filesystem or kernel fault emulation
- power-loss modeling beyond the explicit M0 crash matrix
- probabilistic fuzz campaigns in the main CI lane
- multi-process locking races
- recovery repair or page salvage
- user-configurable failpoint scripting in production binaries

## 10. Implementation Order

1. Add the stable commit phase enum shared by commit code and tests.
2. Add a test-only file adapter interface that can intercept write, grow, and
   sync operations.
3. Implement single-shot scripted faults for write failure, short write, crash,
   and sync uncertainty.
4. Add the phase recorder and expose it to tests.
5. Build deterministic fixtures that can assert old and new metadata tuples.
6. Add reopen helpers that always use normal `open()` rather than a recovery
   shortcut.
7. Add Class A tests for tail growth, data write, and data sync failures.
8. Add Class B tests for metadata short write, torn write, crash-after-write,
   and metadata sync uncertainty.
9. Add Class C success tests that prove publication happens only after confirmed
   metadata sync.
10. Add deterministic corruption-mutation tests for metadata, root, and
    freelist pages.
11. Partition the suite into fast PR coverage and slower exhaustive coverage if
    needed.

## 11. Acceptance

This feature is complete for M4 when:

- the test harness injects faults below the storage engine and above raw OS file
  calls
- the harness and commit path share the stable phase names
  `reserve_tail_pages`, `write_data_pages`, `sync_data_pages`,
  `build_metadata`, `write_metadata`, `sync_metadata`, and
  `publish_metadata`
- deterministic short writes are supported for normal pages and metadata at
  offset `0`, a middle offset, and `page_size - 1`
- crash simulation can stop before and after each durable boundary and reopen
  the database through normal `open()`
- sync simulation distinguishes confirmed failure before issue from uncertain
  post-issue failure
- every Class A and Class B test asserts that the current process stayed on old
  metadata after failure
- reopen assertions verify selected slot and selected generation whenever slot
  behavior matters
- deterministic fixtures provide stable old/new metadata tuples and committed
  key/value expectations
- corruption fixtures can mutate metadata, root, freelist, and arbitrary normal
  pages by role
- production builds cannot enable the fault injection adapter accidentally
- CI coverage keeps explicit tests for Class A, Class B, Class C, and critical
  corruption outcomes
