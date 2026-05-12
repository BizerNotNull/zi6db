# M2 Feature Plan: Benchmark Harness And Randomized Correctness

This plan defines the concrete M2 implementation work for deterministic
randomized correctness tests and a small benchmark harness. It refines the M2
implementation plan while keeping performance measurement separate from
performance tuning.

The shared purpose is repeatability. Random tests must reproduce the exact
operation sequence from a seed, and benchmarks must report enough parameters to
make local results explainable without turning M2 acceptance into a throughput
target.

## 1. Goal

M2 should include:

- a deterministic random operation test that compares `zi6db` against an
  in-memory sorted reference map
- periodic clean close and reopen during the same random sequence
- structural checker integration after reopen and at deterministic batch
  boundaries
- a benchmark harness for insert, lookup, missing lookup, and replace workloads
- small fixtures that force B+Tree splits quickly and default-page-size fixtures
  that exercise the normal path

These tools should make lookup, mutation, delete-without-rebalance, persistence,
and checker defects visible before later milestones add transactions, crash
faults, cursors, buckets, overflow pages, and freelist reuse.

## 2. Scope

In scope:

- Deterministic operation generation from an explicit seed.
- `put`, `delete`, `get`, and `exists` random correctness coverage.
- Reference comparison against an in-memory sorted map using the same raw-byte
  lexicographic ordering as production.
- Periodic close and reopen of the database file during random tests.
- Checker runs after reopen and after deterministic operation batches.
- Benchmark workloads for sequential insert, random insert, existing point
  lookup, missing point lookup, and mixed replace.
- Reporting operation count, elapsed time, operations per second, file size,
  page count, tree height when cheap, seed, key size, value size, page size, and
  workload name.
- Fixture definitions for fast smoke runs and larger local runs.

Out of scope:

- Absolute performance targets.
- Cross-database benchmark comparisons.
- Range scans, prefix scans, cursors, buckets, overflow values, freelist reuse
  tuning, or cross-process concurrency.
- Power-loss fault injection or crash-recovery proof.
- Public property-testing framework design beyond the M2 deterministic test
  harness.

## 3. Deterministic Random Map Test

The randomized correctness test should maintain two states:

- the database under test, opened on a temporary database file
- an in-memory reference map keyed by owned byte arrays and ordered by the shared
  unsigned-byte lexicographic comparator

Each generated operation is applied to both states. After every operation, the
observable result must match:

- `put` updates or inserts the exact generated value in both states
- `delete` returns `true` only when the key existed in both states
- `get` returns `null` for absent keys and byte-for-byte value equality for
  present keys
- `exists` returns the same boolean as the reference map

The test must accept a fixed seed in source control and print or otherwise expose
that seed on failure. Additional seeds may be used in local-only runs, but CI
must include at least one stable seed so failures are reproducible.

The reference map must not rely on host string ordering. It should compare raw
`[]const u8` keys with the same comparator used by search, mutation, and the
checker.

## 4. Operation Generation

Operation generation should be deterministic from `(seed, operation_index,
fixture)`.

Recommended operation mix for the primary random test:

| Operation | Weight | Notes |
| --- | ---: | --- |
| `put` | 45% | insert missing keys and replace existing keys |
| `delete` | 20% | include present, deleted, and never-inserted keys |
| `get` | 25% | probe present, deleted, and never-inserted keys |
| `exists` | 10% | mirror `get` key selection without value allocation |

Key selection should use three pools:

- present keys sampled from the reference map
- previously deleted keys retained in a tombstone probe list
- newly generated keys that may never have been inserted

Key bytes should include:

- fixed-width numeric keys for predictable split patterns
- variable-length byte keys
- prefix-related keys such as `a`, `aa`, and `ab`
- binary keys containing `0x00`
- high-bit bytes such as `0x80` through `0xff`
- the empty key if the M2 API supports it

Value bytes should include:

- empty values
- fixed-size values for stable page-fill behavior
- variable-size values
- replacements that shrink, grow, and keep the same encoded size
- values near the inline entry limit, excluding overflow pages

Generated operations should be recorded in failure output as a compact replay
prefix: seed, fixture name, operation index, operation kind, key bytes in hex,
and value length or value hash. Full values do not need to be printed unless the
test framework already has a safe concise representation.

## 5. Reopen Cadence

Randomized tests should verify persistence throughout the sequence, not only at
the end.

Use a deterministic reopen cadence:

- close and reopen after operation `0` if the test created an initial empty file
- close and reopen every fixed batch, such as every `64` operations in smoke
  fixtures
- close and reopen immediately after selected structural events when observable,
  such as the first root split or after a large delete batch
- close and reopen once at the end before final verification

After each reopen:

1. Open the same file through the normal M1/M2 open path.
2. Compare every key in the reference map with point `get`.
3. Probe a bounded sample of deleted and never-inserted keys.
4. Run the full structural checker.
5. Continue the remaining generated operations with the reopened handle.

Reopen cadence must not depend on wall-clock time. It must be derived from the
fixture and operation index so failures replay exactly.

## 6. Checker Integration

The randomized test should use the M2 structural checker as a correctness oracle
for tree shape, not as a replacement for map comparison.

Required checker calls:

- after creating or opening the initial empty database
- after every deterministic reopen
- after every operation batch, such as every `64` operations in smoke fixtures
- after the final operation and final reopen

The checker must run in full-tree mode for randomized correctness tests. A
root-only check is not sufficient because the random workload is intended to
find stale separator and child-range bugs created by split and no-rebalance
delete paths.

If the checker reports a corruption finding, the test failure should include the
seed, fixture, operation index, and stable `CheckCode` values.

## 7. Benchmark Harness

The benchmark harness should be a small executable or test entry point, for
example `bench/m2_kv.zig` or `test/bench_m2.zig`, depending on the source layout
that exists when M2 is implemented.

Required workloads:

| Workload | Description |
| --- | --- |
| `insert-sequential` | insert fixed-size keys in increasing lexicographic order |
| `insert-random` | insert fixed-size keys generated from a seed |
| `lookup-existing` | point lookup keys known to exist |
| `lookup-missing` | point lookup keys known to be absent |
| `replace-mixed` | repeatedly replace values for existing keys with same, smaller, and larger values |

Each workload should:

- create a fresh temporary database unless explicitly configured to reuse a
  prepared fixture
- use one unnamed key space
- use inline values only
- accept seed, operation count, page size, key size, value size, and output mode
  parameters
- run enough setup to make lookup workloads meaningful before timing begins
- avoid including database creation in the timed region unless the workload name
  explicitly says so
- close the database at the end and report final file size

Benchmark output should be stable and script-friendly. A line-oriented text
format is acceptable for M2; JSON may be added if convenient, but is not
required.

## 8. Metrics

Benchmarks should report at least:

- workload name
- operation count
- elapsed time
- operations per second
- database file size in bytes
- metadata page count
- page size
- key size
- value size
- random seed
- durability mode if configurable

Report these metrics when available without complicating the benchmark path:

- tree height from `CheckReport`
- reachable leaf page count
- reachable branch page count
- checker elapsed time outside the workload timing region

The harness must not fail because throughput is below a threshold. Performance
numbers are diagnostic signals for developers, not M2 release gates.

## 9. Fixtures

Use named fixtures so CI and local benchmark runs are easy to discuss.

Required correctness fixtures:

| Fixture | Purpose |
| --- | --- |
| `tiny-page-smoke` | small page size, small operation count, forces leaf and root splits quickly |
| `default-page-smoke` | default page size, bounded operation count, exercises normal M0 page shape |
| `delete-heavy-routing` | higher delete weight to stress stale separators and empty leaves |
| `binary-keys` | emphasizes embedded zero bytes, high-bit bytes, prefixes, and empty values |

Required benchmark fixtures:

| Fixture | Purpose |
| --- | --- |
| `bench-smoke` | small operation count that proves the harness runs in CI or locally |
| `bench-local` | larger deterministic local run for rough throughput comparison between commits |

Fixture definitions should include:

- page size
- operation count
- reopen interval for correctness tests
- random seed
- key size policy
- value size policy
- operation mix for randomized correctness fixtures

Small-page fixtures must stay within the M0 allowed page-size range. They should
force splits quickly without requiring non-M2 behavior such as overflow pages.

## 10. CI And Smoke Boundaries

CI should run correctness coverage, not long performance experiments.

Recommended CI/smoke boundary:

- run deterministic random map tests with `tiny-page-smoke` and
  `default-page-smoke`
- run at least one delete-heavy fixture if runtime remains reasonable
- run the benchmark harness in `bench-smoke` mode only to prove it starts,
  completes, and reports required metrics
- do not assert absolute operations per second
- keep operation counts bounded so the full M2 test suite remains practical
- print seeds and fixture names for every randomized failure

Longer benchmark fixtures should be opt-in local commands documented by the M2
implementation PR. They should not be required for every pull request.

## 11. Non-Goals

This feature must not expand M2 to include:

- randomized power-loss testing
- fault-injection scheduling
- multi-operation transactions
- snapshot isolation testing
- cursor or range-scan benchmarks
- overflow value benchmarks
- freelist reuse benchmarks
- compaction, vacuum, backup, or statistics benchmarks
- claims about production latency, tail latency, or durability guarantees
- benchmark comparisons against SQLite, LMDB, RocksDB, BoltDB, or other engines

## 12. Implementation Steps

1. Add a deterministic PRNG wrapper used only by tests and benchmarks.
2. Add raw-byte key and value generation helpers with hex-friendly failure
   formatting.
3. Add an in-memory sorted reference map using the shared comparator.
4. Implement the random operation generator with stable weighted choices.
5. Implement the random map test loop for `put`, `delete`, `get`, and `exists`.
6. Add deterministic reopen cadence and final reopen verification.
7. Wire full-tree checker calls into random test batch boundaries and reopen
   points.
8. Add named correctness fixtures for tiny page, default page, delete-heavy, and
   binary-key coverage.
9. Add the benchmark harness entry point and required workloads.
10. Add metrics collection and stable reporting.
11. Add `bench-smoke` coverage that verifies the harness completes without
    asserting throughput.
12. Document local test and benchmark commands in the implementation PR.

## 13. Acceptance

This feature is complete for M2 when:

- deterministic randomized correctness tests compare every operation against an
  in-memory sorted reference map
- random operation generation is fully determined by seed, operation index, and
  fixture
- generated keys cover fixed-width, variable-length, prefix, binary, high-bit,
  deleted, never-inserted, and present-key probes
- generated values cover empty, fixed-size, variable-size, shrinking
  replacement, growing replacement, and near-inline-limit cases
- clean close and reopen occurs at deterministic intervals and at final
  verification
- after each reopen, all reference-map keys and bounded absent-key probes match
  the database
- the full structural checker runs after initial open, after reopen, at batch
  boundaries, and at final verification
- checker failures include stable finding codes plus seed, fixture, and
  operation index context
- benchmark workloads cover sequential insert, random insert, existing lookup,
  missing lookup, and mixed replace
- benchmark reports include operation count, elapsed time, ops/sec, file size,
  page count, page size, key size, value size, and seed
- fixtures include small-page split-heavy coverage and default-page smoke
  coverage
- CI runs bounded randomized correctness tests and a benchmark smoke run without
  enforcing absolute throughput
- longer benchmark runs remain opt-in local tools
- the harness stays limited to one unnamed key space and inline values
- the work leaves clear seams for M3 transactions, M4 fault injection, M5
  cursors, M6 overflow/freelist reuse, and M7 verify tooling
