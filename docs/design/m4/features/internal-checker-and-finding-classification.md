# M4 Feature Plan: Internal Checker And Finding Classification

## Goal

M4 expands the earlier structural checker into a file-level internal checker that
understands crash-recovery metadata selection, critical page validation, and
selected-state traversal. The checker remains internal in M4, but its result
shape must already be stable enough that M7 can expose it as the basis for a
public read-only verify API and CLI without redesigning the core finding model.

This feature exists to make crash, corruption, and recovery behavior testable at
the same level of precision required by the M0 design freeze. It must explain
why a database can open, why one metadata slot was rejected, and why a selected
database state is valid, suspicious, or corrupted.

## Scope

In scope:

- Return structured findings with stable severity and classification codes.
- Return explicit per-slot validation results for metadata slot A and slot B.
- Re-run M4 metadata selection rules and report the selected-state outcome.
- Validate the selected root page and selected freelist page as critical pages.
- Traverse the selected B+Tree and detect selected-state corruption.
- Distinguish critical corruption from informational leftover pages.
- Keep traversal scoped to the selected metadata snapshot unless an explicit
  non-critical physical-page scan mode is requested for tests.
- Preserve enough context for M7 verify reporting, including slot, page, owner,
  generation, and `txn_id` data.

Out of scope:

- A public `verify()` API or CLI command. That is M7 scope.
- Repair, salvage, page relocation, or free-space inference from unreachable
  pages.
- Cross-process lock findings. Those are M7 scope.
- Full overflow-chain and full freelist ownership validation beyond the M4
  critical-path checks needed for selected metadata, selected root, selected
  freelist, and reachable tree pages.

## Design Principles

- The checker must be read-only and must never repair or rewrite the file.
- The checker must follow the same header, metadata, checksum, and slot
  selection rules as `open()`.
- Findings must be classified by stable machine-readable codes, not only prose.
- A corrupted inactive slot is reportable, but it must not by itself make the
  selected database state fail when the other slot validates and wins.
- Selected-state corruption is more severe than inactive-slot corruption because
  it determines whether public `open()` should succeed.
- Informational leftover pages from failed commits must remain non-authoritative
  and must never be reclassified as reusable or repaired by the checker.

## Recommended API Shape

M4 may keep the checker internal, but the implementation should converge on a
shape close to:

```zig
pub fn check(db: *DB, options: CheckOptions) !CheckReport;
pub fn checkFile(file: *StorageFile, options: CheckOptions) !CheckReport;
```

`check(db, ...)` should use the currently selected in-memory metadata snapshot
unless the caller supplies an override for recovery tests. `checkFile(...)`
should support startup-style validation before a `DB` instance is fully
published.

## Check Options

```zig
const CheckOptions = struct {
    mode: CheckMode = .selected_state,
    metadata_override: ?MetadataSnapshot = null,
    include_informational: bool = true,
    scan_unreachable_within_page_count: bool = false,
    max_findings: ?usize = null,
};

const CheckMode = enum {
    metadata_only,
    selected_state,
    selected_state_with_unreachable_scan,
};
```

Rules:

- `metadata_only` validates the header, slot A, slot B, and selection outcome
  without traversing the selected tree.
- `selected_state` validates the selected critical pages and traverses only
  pages reachable from the selected metadata snapshot.
- `selected_state_with_unreachable_scan` is a deeper internal test mode that may
  inspect non-critical pages below selected `page_count`, but it must still keep
  selected-state findings separate from non-authoritative physical leftovers.
- `metadata_override` is for deterministic recovery tests and must not let the
  checker bypass normal slot-validation rules.
- `max_findings` may truncate traversal, but the report must mark itself
  incomplete.

## Report Shape

The report should separate file-level outcome, per-slot outcome, and selected
state outcome.

```zig
const CheckReport = struct {
    ok: bool,
    complete: bool,
    mode: CheckMode,
    selected_slot: ?MetadataSlotId,
    selected_generation: ?u64,
    selected_txn_id: ?u64,
    page_size: ?u32,
    page_count: ?u64,
    slot_a: SlotCheckResult,
    slot_b: SlotCheckResult,
    selected_state: SelectedStateResult,
    findings: []CheckFinding,
};

const SlotCheckResult = struct {
    slot: MetadataSlotId,
    physical_page_id: u64,
    valid: bool,
    generation: ?u64,
    txn_id: ?u64,
    invalidates_open_by_itself: bool,
    findings_range: FindingRange,
};

const SelectedStateResult = struct {
    has_selected_state: bool,
    selected_slot: ?MetadataSlotId,
    reachable_tree_pages: u64,
    reachable_branch_pages: u64,
    reachable_leaf_pages: u64,
    findings_range: FindingRange,
};

const FindingRange = struct {
    start: usize,
    count: usize,
};
```

Rules:

- `ok` is true only when there are no findings with severity `corruption` or
  `incompatible` that apply to the selected-state result or make metadata
  selection impossible.
- `slot_a.valid` and `slot_b.valid` describe the slots independently, even when
  the overall file cannot open.
- `invalidates_open_by_itself` is true only when the slot would be the selected
  slot if the other slot did not exist or was also invalid.
- `selected_state.has_selected_state` is false when no valid slot can be
  selected.

## Finding Shape

```zig
const CheckFinding = struct {
    severity: CheckSeverity,
    code: CheckCode,
    scope: FindingScope,
    metadata_slot: ?MetadataSlotId = null,
    page_id: ?u64 = null,
    page_type: ?PageType = null,
    owner: ?FindingOwner = null,
    generation: ?u64 = null,
    txn_id: ?u64 = null,
    message: []const u8,
};

const CheckSeverity = enum {
    ok,
    info,
    warning,
    corruption,
    incompatible,
};

const FindingScope = enum {
    file,
    slot_a,
    slot_b,
    selected_state,
    unreachable_scan,
};

const FindingOwner = enum {
    header,
    metadata,
    root,
    freelist,
    btree_branch,
    btree_leaf,
    physical_tail,
};
```

The implementation should define stable `CheckCode` values for at least:

- `invalid_header`
- `incompatible_format`
- `invalid_metadata_slot`
- `metadata_checksum_mismatch`
- `metadata_equal_generation`
- `metadata_txn_regression`
- `critical_page_out_of_range`
- `critical_page_checksum_mismatch`
- `invalid_page_type`
- `duplicate_reachable_page`
- `reachable_page_out_of_range`
- `freelist_reserved_page`
- `ignored_unreachable_page`
- `unreachable_page_within_page_count`
- `incomplete_check`

## Severity Model

M4 should keep the severity ladder aligned with the M4 implementation plan and
compatible with the M7 verify handoff.

| Severity | Meaning in M4 | Public boundary implication |
| --- | --- | --- |
| `ok` | explicit success marker only when useful in slot summaries | none |
| `info` | allowed leftover or diagnostic-only state | does not fail `open()` |
| `warning` | slot-local invalidity or suspicious non-authoritative state | may fail `open()` only if no valid slot remains or selection fails |
| `corruption` | selected-state corruption or unrecoverable metadata conflict | maps to `Corruption` |
| `incompatible` | unsupported version or required feature set | maps to `IncompatibleVersion` |

Rules:

- A metadata checksum mismatch in the inactive losing slot is normally a
  `warning`, not immediate selected-state `corruption`.
- Equal valid generations across slots are always `corruption` because selection
  must fail.
- A selected root or selected freelist checksum mismatch is always
  `corruption`.
- Ignored pages beyond selected `page_count` are always `info`.
- Unreachable pages inside selected `page_count` are `info` or `warning` only in
  the explicit unreachable-scan mode; they are not part of the normal selected
  traversal contract in M4.

## Per-Slot Result Rules

Each metadata slot must be checked independently before any generation
comparison.

Per-slot validation must record:

- physical slot location and expected `slot_id`
- metadata checksum status
- page size and format compatibility fields
- reserved-field and required-feature validation
- `page_count`, `root_page_id`, and `freelist_page_id` range checks
- selected root page checksum and type checks
- selected freelist page checksum and type checks
- decoded `generation` and `txn_id` when available

Per-slot result semantics:

- A slot with a bad checksum, wrong `slot_id`, incompatible required feature
  flag, out-of-range root/freelist pointer, or invalid referenced critical page
  is `valid = false`.
- Slot-local invalidity must remain attached to `slot_a` or `slot_b` even when
  the other slot allows a successful open.
- If both slots are invalid, the report must show both slot results and a
  selected-state failure.
- If both slots are valid but generations tie, both slot results remain valid
  and the selected-state outcome records `metadata_equal_generation`.
- If the newer generation has a lower `txn_id` than the older valid slot, both
  slot results remain individually valid and the selected-state outcome records
  `metadata_txn_regression`.

## Critical Vs Informational Findings

M4 must clearly separate critical corruption from leftover non-authoritative
state.

Critical findings:

- invalid header magic or checksum
- incompatible major version or unsupported required feature flags
- no valid metadata slot
- equal valid metadata generations
- decreasing `txn_id` across otherwise valid generations
- selected root out of range, wrong type, unreadable bytes, or checksum mismatch
- selected freelist out of range, wrong type, unreadable bytes, or checksum
  mismatch
- duplicate reachable tree page
- reachable selected-tree page outside selected `page_count`

Informational findings:

- ignored tail pages beyond selected `page_count` after failed or partial commit
- corrupted inactive slot when the active slot remains valid and selection is
  deterministic
- optional unreachable-page scan reports for non-critical pages not referenced
  by the selected metadata

Rules:

- Informational findings must never change which metadata slot is selected.
- Informational findings must never upgrade a successful `open()` into failure.
- Critical findings must identify whether they invalidate slot A, slot B, or the
  selected state.

## Traversal Scope

The checker must not silently drift from M4 recovery rules by scanning arbitrary
pages and treating them as authoritative.

Normal selected-state traversal:

1. Validate the header.
2. Validate slot A and slot B independently.
3. Re-run metadata selection exactly as `open()` does.
4. If no selected slot exists, stop selected traversal and report the selection
   failure.
5. Read and validate the selected root and selected freelist as critical pages.
6. Traverse the selected B+Tree from the selected root.
7. Track visited reachable page ids and fail on duplicates.
8. Ensure reachable page ids stay within selected `page_count`.
9. Ensure reachable page types are allowed for their traversal location.

Traversal limits:

- The default M4 checker must only traverse the selected B+Tree and selected
  critical pages.
- It must not interpret pages outside selected reachability as live data.
- It may note physical pages beyond selected `page_count` as
  `ignored_unreachable_page` informational findings without decoding them deeply.
- It may scan unreachable pages below selected `page_count` only in the explicit
  deeper test mode, and those results must stay in `unreachable_scan` scope.

This scope boundary preserves M0's rule that recovery trusts only the selected
metadata snapshot and does not reclaim or reinterpret failed-commit leftovers.

## Classification Mapping

The checker should freeze a small mapping from finding code to severity and
public-boundary meaning so M7 can reuse the codes directly.

| Check code | Default severity | Selected-state meaning |
| --- | --- | --- |
| `invalid_header` | `corruption` | `open()` must fail |
| `incompatible_format` | `incompatible` | `open()` must fail |
| `invalid_metadata_slot` | `warning` | slot is excluded from selection |
| `metadata_checksum_mismatch` | `warning` | slot is excluded from selection |
| `metadata_equal_generation` | `corruption` | selection fails |
| `metadata_txn_regression` | `corruption` | selection fails |
| `critical_page_out_of_range` | `corruption` | referenced slot is invalid |
| `critical_page_checksum_mismatch` | `corruption` | referenced slot is invalid |
| `invalid_page_type` | `corruption` | slot invalid or selected traversal fails |
| `duplicate_reachable_page` | `corruption` | selected traversal fails |
| `reachable_page_out_of_range` | `corruption` | selected traversal fails |
| `freelist_reserved_page` | `corruption` | selected traversal fails |
| `ignored_unreachable_page` | `info` | report only |
| `unreachable_page_within_page_count` | `warning` | report only unless a later milestone promotes it |

The implementation may add more codes, but it should not rename these baseline
codes once M4 tests depend on them.

## Test Cases

M4 should add checker-focused tests in addition to the broader crash and
corruption matrix.

### Structured Findings

- clean database returns `ok = true` with zero corruption findings
- invalid inactive metadata slot produces a slot-scoped finding while selected
  state remains valid
- equal valid generations produce `metadata_equal_generation`
- newer generation with lower `txn_id` produces `metadata_txn_regression`
- selected root checksum mismatch produces a selected-state corruption finding
- selected freelist wrong page type produces a selected-state corruption finding

### Per-Slot Results

- slot A valid and slot B invalid reports `slot_a.valid = true`,
  `slot_b.valid = false`, and selected slot A
- slot B valid and slot A invalid reports `slot_b.valid = true`,
  `slot_a.valid = false`, and selected slot B
- both invalid reports no selected state and preserves both slot finding ranges
- both valid with different generations reports the newer slot as selected while
  preserving the losing slot result

### Severity And Classification

- inactive slot checksum mismatch is `warning`
- ignored tail page beyond selected `page_count` is `info`
- selected reachable duplicate page is `corruption`
- unsupported required feature flags are `incompatible`

### Traversal Scope

- duplicate reachable selected-tree page is detected during selected traversal
- reachable page id equal to or greater than selected `page_count` is
  `corruption`
- non-critical page beyond selected `page_count` is not treated as live
  corruption
- unreachable page below selected `page_count` is reported only when the deeper
  unreachable-scan mode is enabled

### Crash And Recovery Integration

- Class A commit failure leaves old selected state valid and may produce ignored
  tail-page info findings
- Class B ambiguous metadata outcome preserves old in-process metadata while a
  reopen checker report deterministically explains which slot won
- torn inactive metadata slot is reported as slot-local invalidity and does not
  fail open when the older slot remains valid

## M7 Verify Handoff

M4 must deliberately leave a seam for M7 rather than a dead-end internal test
utility.

M4 should hand off:

- stable finding severities and baseline codes
- explicit slot-scoped, selected-state-scoped, and unreachable-scan-scoped
  findings
- per-slot summaries with `generation`, `txn_id`, and validity flags
- a report shape that can grow into M7 `VerifyReport` without dropping fields
- selected-state traversal helpers that M7 can extend with overflow, bucket, and
  full freelist ownership checks
- informational classification for ignored tail pages so M7 verify can preserve
  the same non-fatal behavior

M7 may extend this with:

- public `verify()` and CLI output
- JSON serialization of findings
- overflow-chain, bucket, freelist ownership, and lock-related issue kinds
- configurable deep unreachable-page scanning
- richer owner context and truncation reporting

M4 must not require M7 to reinterpret or rename M4 finding codes just to expose
them publicly.

## Acceptance

This feature is complete for M4 when:

- the internal checker returns structured findings with stable severity and code
  fields
- the checker reports independent results for metadata slot A and slot B
- selected-state success or failure is separated from slot-local invalidity
- inactive-slot corruption can be reported without making a valid selected state
  fail
- critical selected root and selected freelist failures are classified as
  corruption
- ignored unreachable tail pages are classified as informational only
- traversal is scoped to the selected metadata snapshot by default
- deeper unreachable-page inspection is opt-in and remains non-authoritative
- checker tests cover structured findings, per-slot outcomes, severity mapping,
  traversal boundaries, and crash-recovery interactions
- the report shape and classification codes provide a direct handoff path to the
  M7 verify API and CLI
