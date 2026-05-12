# M7 Feature Plan: CI Diagnostics And Benchmark Baselines

This plan defines the concrete M7 implementation work for cross-platform CI,
diagnostic artifact capture, contextual error reporting, and benchmark
baselines. It refines the M7 implementation plan into one feature slice that
turns production-hardening requirements into stable workflows and reviewable
release signals.

The shared purpose is explainability. CI must prove supported-platform coverage,
diagnostic outputs must make failures actionable without exposing user payload
bytes, and benchmark baselines must provide repeatable regression signals
without pretending that hosted runners are authoritative performance gates.

## 1. Goal

M7 should include:

- a required CI matrix covering Linux, macOS, and Windows
- deterministic upload of failure artifacts and benchmark smoke artifacts
- stable contextual error reporting for `open`, `verify`, `commit`, `backup`,
  `compact`, `lock`, and `sync`
- a benchmark harness expansion that covers operational M7 workloads in
  addition to core KV paths
- baseline benchmark reports for one representative local machine and bounded CI
  smoke runs
- a documented non-gating policy for timing-based benchmark results until a
  later performance-threshold milestone exists

These deliverables should make platform regressions, corruption findings,
locking failures, backup failures, and performance shifts visible early while
preserving the frozen M0 rules and the M6 handoff seams.

## 2. Scope

In scope:

- GitHub Actions or equivalent hosted CI workflows for Linux, macOS, and
  Windows.
- Debug and release-safe validation coverage where practical within CI budget.
- Required CI steps for formatting, build, unit tests, integration tests,
  corruption tests, locking tests, backup/restore tests, verify tests, and
  benchmark smoke.
- Upload policy for logs, verify JSON reports, corrupted test fixtures,
  benchmark smoke output, and crash-edge diagnostics.
- Structured error-message context rules shared across CLI, tests, and internal
  diagnostics.
- Benchmark workload definitions, fixture classes, reported metrics, and
  baseline-report metadata.
- Non-gating timing policy for pull-request CI and a separate path for longer
  scheduled or manual benchmark runs.
- Acceptance criteria for CI coverage, artifact quality, diagnostic wording, and
  baseline completeness.

Out of scope:

- Introducing a new public error taxonomy beyond the M0 categories.
- Absolute performance targets or release-blocking throughput thresholds.
- Cross-database comparisons against other engines.
- Hosted CI claims about power-loss proof, every filesystem, or every antivirus
  and network-drive environment.
- Reworking the on-disk format, metadata rules, or freelist semantics.

## 3. CI Matrix

The required CI matrix for M7 should use the latest stable hosted images for:

- Linux
- macOS
- Windows

Each platform should run enough coverage to validate correctness-critical M7
behavior, not only compilation.

Required matrix dimensions:

| Dimension | Required coverage |
| --- | --- |
| Operating system | Linux latest stable, macOS latest stable, Windows latest stable |
| Build mode | `Debug` and `ReleaseSafe` where runtime budget remains practical |
| Test grouping | formatting, build, unit, integration, corruption/verify, locking, backup/restore, crash-edge, benchmark smoke |
| Trigger class | pull request, protected branch push, optional scheduled/manual long benchmark jobs |

Minimum required pull-request jobs:

| Job | Purpose |
| --- | --- |
| `fmt` | run `zig fmt --check` or equivalent |
| `build` | run `zig build` on each supported OS |
| `test-core` | run unit and integration tests |
| `test-diagnostics` | run verify, corruption, backup/restore, and error-context tests |
| `test-locking` | run multi-process locking and reader-tracking tests |
| `bench-smoke` | run bounded benchmark fixtures and upload outputs |

Platform-specific requirements:

- Windows jobs must include path, rename, sharing-mode, and `LockFileEx`
  behavior coverage.
- POSIX jobs must include `fcntl` byte-range lock coverage.
- Any skipped correctness-critical test must record a reason, owner, and linked
  issue before `v1.0.0`.
- Crash and fault-injection jobs must use deterministic injected failure phases
  and IPC barriers rather than sleep-based races.

## 4. Artifact Policy

CI artifacts are part of the diagnostic contract. When a correctness test or
smoke benchmark fails, reviewers should have enough preserved context to inspect
the failure without rerunning locally.

Required artifact classes:

| Artifact | Required contents |
| --- | --- |
| verify report | human summary plus JSON findings when verify-related tests fail |
| corruption fixture copy | the temporary mutated database file that triggered the failure when safe to retain |
| crash-edge log | injected failure phase, expected outcome, observed selected metadata slot, and reopen result |
| locking log | process roles, barrier sequence, lock acquisition outcome, and public error category |
| benchmark smoke report | workload name, fixture, seed when applicable, metrics, Zig version, OS, build mode, and git commit |
| test command summary | exact command and job metadata used to produce the artifact |

Artifact retention rules:

- Pull-request artifacts should be retained long enough for normal review and
  follow-up triage.
- Protected-branch and scheduled benchmark artifacts should be retained longer
  than ordinary pull-request artifacts when they are used for baseline
  comparison.
- Artifact naming must be deterministic and include OS, build mode, job name,
  and test group.

Safety and privacy rules:

- Artifacts must not include raw user key or value payload bytes by default.
- Verify JSON and logs may include page id, page type, metadata slot,
  generation, `txn_id`, logical owner, expected value, and actual value for
  structural fields.
- If an opt-in debug mode ever permits payload excerpts, the output must be
  explicit, bounded, and excluded from default CI uploads.
- Temporary files that are not needed for diagnosis should be removed before
  artifact packaging.

## 5. Error-Message Context

M7 should standardize human-facing diagnostics around contextual fields instead
of expanding the public error surface.

Required context fields:

| Context class | Required examples |
| --- | --- |
| operation | `open`, `verify`, `commit`, `backup`, `compact`, `lock`, `sync` |
| path | database path, destination path, temporary path when relevant |
| structure | page id, page type, metadata slot, root/freelist role, logical owner |
| ordering | generation, `txn_id`, expected ordering, selected slot |
| failure class | `Corruption`, `IncompatibleVersion`, `LockConflict`, `WriteConflict`, `ReadOnlyTransaction`, `InvalidState`, `IoError` |
| authority note | explicit note that reopen/recovery is authoritative after ambiguous commit outcomes |

Required wording rules:

- Public text must distinguish corruption from lock conflict and generic I/O
  failure.
- Checksum failures on critical structures must present as corruption in public
  text while preserving finer internal or verify issue kinds.
- Lock conflicts must never be reported as plain `IoError`.
- Diagnostics must identify which metadata slot remained selected when another
  slot failed validation.
- Verify and backup failures must identify the phase in which the failure
  occurred.
- Error text must stay concise enough for CLI output and CI logs while still
  exposing the fields above.

Representative message shapes:

- `open failed: metadata slot B checksum mismatch at page 2; selected slot A remains valid`
- `verify failed: freelist page 42 contains page 107 also reachable from bucket "users"`
- `lock conflict: database already has an active writer lock`
- `backup failed while syncing destination file "backup.zi6db": no space left on device`
- `commit outcome uncertain after metadata sync failure; close handle and reopen database to determine authoritative state`

Tests for error-message context should verify the presence of operation and
structural fields without pinning exact OS-native wording.

## 6. Benchmark Workloads And Metrics

M7 benchmarks should extend earlier harness work so operational features have
visibility alongside core KV paths.

Required workloads:

| Workload | Description |
| --- | --- |
| `create-and-load` | create a fresh database and load an initial dataset |
| `insert-sequential` | insert keys in increasing lexicographic order |
| `insert-random` | insert keys from a deterministic seed |
| `lookup-point` | point lookups for existing keys |
| `scan-range` | bounded ordered scans across a key interval |
| `scan-prefix` | prefix-filtered scans over clustered keys |
| `update-existing` | replace values for already present keys |
| `delete-existing` | delete keys known to exist |
| `overflow-large-value` | insert and read values large enough to use overflow pages |
| `txn-mixed-read-write` | mixed transactional workload with reads and writes |
| `backup-throughput` | snapshot backup throughput for a prepared dataset |
| `verify-throughput` | verify throughput on a clean prepared dataset |
| `compact-throughput` | optional compaction throughput when compaction ships |

Required benchmark fixtures:

| Fixture | Purpose |
| --- | --- |
| `bench-smoke` | bounded CI fixture that proves workloads run and emit metrics |
| `bench-baseline-small` | local baseline fixture with practical runtime for review |
| `bench-baseline-large` | larger local or scheduled fixture for trend comparison |
| `bench-overflow` | focuses on large values and overflow-chain behavior |
| `bench-operational` | focuses on verify and backup throughput on an already loaded database |

Required reported metrics:

- workload name
- fixture name
- operation count or bytes processed, depending on workload
- elapsed time
- operations per second or bytes per second
- latency percentiles when the harness supports them
- database file size in bytes
- page count
- free page count
- overflow page count
- bytes written when measurable
- durability mode
- page size
- dataset size and key/value distribution
- warm or cold run classification
- operating system and filesystem label when known
- Zig version
- build mode
- exact zi6db git commit
- whether verify, backup, or compaction used deep scanning where relevant

Reporting rules:

- Output should be script-friendly and stable across runs.
- JSON output is preferred for baseline archival and CI artifact parsing.
- Benchmark datasets must be synthetic or otherwise safe to publish in CI
  artifacts.
- Comparisons against previous baseline artifacts are diagnostic only unless a
  later threshold policy explicitly upgrades them to gates.

## 7. Non-Gating Policy

Timing results in M7 are release signals, not correctness gates.

Pull-request CI policy:

- `bench-smoke` must fail only on harness failure, correctness failure, missing
  required metrics, or obviously invalid output such as zero successful
  operations.
- Pull-request CI must not fail solely because ops/sec, latency, or throughput
  is worse than a previous run.
- Benchmark smoke must stay small enough that it does not dominate CI runtime or
  hide correctness failures behind flaky timing variance.

Baseline comparison policy:

- One representative local-machine baseline report is required for M7
  completion.
- CI smoke numbers should be archived for drift observation but treated as
  non-authoritative because hosted runners are noisy.
- Longer soak, stress, or large-dataset benchmark runs may execute on scheduled
  or manual workflows instead of every pull request.
- If a benchmark shows a suspicious regression, the correct response is
  investigation and confirmation, not immediate automatic failure, unless the
  regression also indicates correctness breakage.

Documentation requirements:

- The baseline report must state hardware class, OS, filesystem, Zig version,
  build mode, durability mode, page size, dataset size, and warm/cold status.
- CI documentation must state explicitly that hosted-runner timing is non-gating
  for M7.
- Any future move to performance thresholds must be a separate documented policy
  change after variance-handling rules exist.

## 8. Implementation Steps

1. Define the canonical CI job groups and the minimum required platform matrix.
2. Split correctness coverage so formatting, build, core tests, diagnostics
   tests, locking tests, and benchmark smoke have clear ownership in CI.
3. Add deterministic artifact packaging for verify reports, corruption
   fixtures, crash-edge logs, locking logs, and benchmark smoke output.
4. Introduce shared diagnostic-context helpers so operation, path, structural,
   and ordering fields are formatted consistently.
5. Audit public error sites for `open`, `verify`, `commit`, `backup`,
   `compact`, `lock`, and `sync` to ensure they carry the required context.
6. Expand the benchmark harness with range scan, prefix scan, overflow, mixed
   transaction, backup, and verify workloads.
7. Add stable benchmark metadata output including Zig version, OS, build mode,
   durability mode, page size, dataset description, and git commit.
8. Add `bench-smoke` fixtures that bound runtime but still exercise required
   metric paths.
9. Document local and CI benchmark fixture classes, including which ones are
   non-gating and which ones are baseline-producing.
10. Capture at least one representative local-machine baseline report and one CI
    smoke artifact set.
11. Add tests that validate error-message context fields and artifact-generation
    behavior without pinning platform-specific prose.
12. Freeze the M7 non-gating policy in documentation before `v1.0.0`.

## 9. Acceptance

This feature is complete for M7 when:

- CI runs required jobs on Linux, macOS, and Windows
- correctness-critical jobs cover formatting, build, unit/integration,
  corruption/verify, locking, backup/restore, and benchmark smoke
- Windows CI exercises path, rename, sharing-mode, and `LockFileEx` behavior
- POSIX CI exercises `fcntl` byte-range locking behavior
- required failure artifacts are uploaded with deterministic names and useful
  context
- artifact uploads omit raw user payload bytes by default
- public diagnostics for `open`, `verify`, `commit`, `backup`, `compact`,
  `lock`, and `sync` include operation context plus the relevant structural
  fields
- lock conflicts are reported distinctly from corruption and generic I/O
  failures
- ambiguous commit failures explicitly tell callers that reopen/recovery is
  authoritative
- benchmark workloads cover create/load, inserts, point lookup, range scan,
  prefix scan, update, delete, overflow values, mixed transactions, backup, and
  verify, plus compaction if compaction ships
- benchmark reports include workload metadata, key metrics, platform/build
  context, and exact git commit
- at least one representative local baseline report exists and is documented
- CI smoke benchmark artifacts exist and are explicitly documented as
  non-gating for timing regressions
- pull-request benchmark jobs fail only on harness, correctness, or invalid
  output errors, not on throughput drift alone
- any skipped mandatory test records a reason, owner, and linked issue before
  `v1.0.0`
- the feature leaves the M0 public error boundary and on-disk format rules
  unchanged
