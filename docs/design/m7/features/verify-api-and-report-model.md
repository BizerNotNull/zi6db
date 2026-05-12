# M7 Feature Plan: Verify API And Report Model

This document defines the concrete M7 implementation plan for the
`verify API and report model` feature referenced by
[docs/design/m7/m7-implementation-plan.md](/D:/代码D/zi6db/docs/design/m7/m7-implementation-plan.md).
It follows the repository rules in
[AGENTS.md](/D:/代码D/zi6db/AGENTS.md) and builds directly on the internal
checker and finding model from
[docs/design/m4/features/internal-checker-and-finding-classification.md](/D:/代码D/zi6db/docs/design/m4/features/internal-checker-and-finding-classification.md).

The purpose of this feature is to expose a public, read-only verification
surface that can explain selected-state corruption, inactive-slot damage,
optional unreachable-page findings, and setup failures without redesigning the
M4 classification seam. M7 verify must remain diagnostic only: it explains
state precisely, but it never repairs, rewrites, truncates, promotes, or
reclassifies free space.

## 1. Scope

M7 verify is complete when:

- callers can run a public `verify()` API against a database path
- callers can choose bounded deep-scan behavior through stable options
- reports preserve stable issue kinds, severity, and owner context
- the CLI can emit both concise human output and stable JSON
- tool exit status distinguishes setup failure from reportable corruption
- tests prove deterministic truncation, traversal boundaries, and JSON stability

M7 verify does not promise:

- repair, salvage, or in-place mutation
- automatic compaction, truncation, or free-space recovery
- interpretation of unreachable pages as reusable space
- payload dumps of user key/value bytes by default

## 2. Design Principles

- Verify must reuse the same header validation, metadata selection, page
  decoding, overflow validation, and freelist validation rules trusted by
  `open()` and M6 helpers.
- Verify must preserve the M4 distinction between slot-local invalidity,
  selected-state corruption, and non-authoritative physical leftovers.
- Verify must collect multiple findings up to a bounded limit instead of
  stopping at the first selected-state issue.
- Verify JSON must be stable enough for downstream tooling before `v1.0.0`;
  field names and issue-kind strings are part of the public contract.
- Optional deep scans must remain opt-in and must never silently expand the
  normal selected-state trust boundary.

## 3. Public API Shape

Recommended Zig surface:

```zig
pub const VerifyOptions = struct {
    mode: VerifyMode = .selected_state,
    include_informational: bool = true,
    max_issues: u32 = 100,
};

pub const VerifyMode = enum {
    metadata_only,
    selected_state,
    selected_state_with_unreachable_scan,
};

pub const VerifyReport = struct {
    ok: bool,
    complete: bool,
    mode: VerifyMode,
    selected_slot: ?MetadataSlotId,
    selected_generation: ?u64,
    selected_txn_id: ?u64,
    page_size: ?u32,
    page_count: ?u64,
    summary: VerifySummary,
    slot_a: VerifySlotResult,
    slot_b: VerifySlotResult,
    issues: []VerifyIssue,
};

pub fn verify(path: []const u8, options: VerifyOptions) !VerifyReport;
```

`verify()` should operate on a path rather than an already-open mutable `DB`
handle so the API can acquire the read-only lock path and re-run startup-style
selection from disk.

## 4. VerifyOptions

`VerifyOptions` should stay intentionally small and map directly to stable
behavior.

### 4.1 `mode`

`mode` defines traversal scope:

- `metadata_only`: validate the header, both metadata slots, selection outcome,
  and critical setup compatibility only
- `selected_state`: validate the selected metadata snapshot, critical pages,
  reachable B+Tree pages, reachable overflow chains, selected freelist pages,
  and ownership overlap inside selected state
- `selected_state_with_unreachable_scan`: run `selected_state` plus an explicit
  scan of pages below selected `page_count` that are not claimed by the
  selected snapshot

M7 should not keep separate booleans such as `scan_pages` and
`scan_unreachable_pages` if they can produce ambiguous or conflicting
combinations. One traversal mode is easier to document and keep stable.

### 4.2 `include_informational`

- when `true`, `info` findings such as ignored tail pages or optional
  diagnostic notes are included in the report
- when `false`, `info` findings may be omitted from `issues`, but summary
  counters should still preserve whether informational findings were suppressed
- suppressing `info` findings must not affect traversal outcome, issue
  severity, or exit status

### 4.3 `max_issues`

- bounds collected findings for deterministic tool output
- applies across all scopes, not per slot or per traversal phase
- when the limit is hit, `complete = false` and the report must include a stable
  truncation issue kind instead of silently stopping
- traversal order must be deterministic so repeated runs on the same file and
  options produce the same prefix of issues

## 5. Report Model

M7 should extend the M4 checker shape rather than replace it.

```zig
pub const VerifySummary = struct {
    error_count: u32,
    warning_count: u32,
    info_count: u32,
    unreachable_scan_enabled: bool,
    truncated: bool,
};

pub const VerifySlotResult = struct {
    slot: MetadataSlotId,
    physical_page_id: u64,
    valid: bool,
    generation: ?u64,
    txn_id: ?u64,
    invalidates_open_by_itself: bool,
};

pub const VerifyIssue = struct {
    severity: VerifySeverity,
    kind: VerifyIssueKind,
    scope: VerifyIssueScope,
    metadata_slot: ?MetadataSlotId = null,
    page_id: ?u64 = null,
    page_type: ?PageType = null,
    owner: ?VerifyOwner = null,
    related_page_id: ?u64 = null,
    related_owner: ?VerifyOwner = null,
    generation: ?u64 = null,
    txn_id: ?u64 = null,
    pending_free_txn_id: ?u64 = null,
    message: []const u8,
};
```

Required report rules:

- `ok` is `true` only when setup succeeded and there are no `error` issues
- `complete` is `false` when traversal truncates at `max_issues`
- `slot_a` and `slot_b` must preserve per-slot results even when no selected
  state exists
- `summary.unreachable_scan_enabled` must reflect the actual traversal mode so
  tooling does not misread a shallow run as exhaustive proof

## 6. Issue Kinds

M7 should preserve M4 baseline codes where possible and add public verify kinds
only where deeper M6 and M7 coverage requires them.

Minimum stable `VerifyIssueKind` set:

- `invalid_header`
- `incompatible_format`
- `unsupported_required_feature`
- `invalid_metadata_slot`
- `metadata_checksum_mismatch`
- `metadata_equal_generation`
- `metadata_txn_regression`
- `critical_page_out_of_range`
- `critical_page_checksum_mismatch`
- `invalid_page_type`
- `reachable_page_out_of_range`
- `duplicate_reachable_page`
- `btree_order_violation`
- `btree_child_range_violation`
- `leaf_payload_out_of_bounds`
- `freelist_page_checksum_mismatch`
- `freelist_page_sequence_invalid`
- `freelist_reserved_page`
- `freelist_duplicate_reusable_page`
- `freelist_duplicate_pending_page`
- `freelist_reusable_pending_overlap`
- `freelist_reachable_page_overlap`
- `overflow_reference_out_of_range`
- `overflow_page_checksum_mismatch`
- `overflow_page_wrong_type`
- `overflow_chain_cycle`
- `overflow_chain_length_mismatch`
- `overflow_chain_payload_length_mismatch`
- `overflow_chain_duplicate_page`
- `duplicate_page_ownership`
- `unreachable_page_within_page_count`
- `ignored_tail_page`
- `lock_conflict`
- `incomplete_report`

Issue-kind rules:

- names must stay lowercase snake_case in API and JSON output
- once published, kinds must not be renamed; adding new kinds later is allowed
- human CLI prose may evolve, but the stable kind string is the contract for
  automation

## 7. Severity Model

M7 should collapse the M4 ladder into the public verify ladder without changing
the underlying meaning.

```zig
pub const VerifySeverity = enum {
    error,
    warning,
    info,
};
```

Mapping rules:

- M4 `corruption` and `incompatible` map to M7 `error`
- M4 `warning` remains M7 `warning`
- M4 `info` remains M7 `info`
- M4 `ok` does not become a public issue; success is represented by an empty
  issue list and `ok = true`

Severity expectations:

- `error`: corruption, incompatible format, unsupported required features,
  metadata selection failure, critical selected-state violations, or reportable
  setup conflicts such as `lock_conflict`
- `warning`: suspicious but non-authoritative findings, typically unreachable
  pages inside selected `page_count` discovered only in the explicit deeper scan
- `info`: ignored tail pages beyond selected `page_count`, inactive-slot damage
  when another slot still wins deterministically, and other diagnostic-only
  notes

Public-boundary rules:

- `error` findings make verify fail
- `warning` findings do not by themselves fail verify
- `info` findings do not affect success or exit status

## 8. JSON Stability

Machine-readable output is a required part of the feature, not an optional
debug convenience.

Required JSON contract:

- top-level field names must be documented and stable before `v1.0.0`
- `severity`, `kind`, `scope`, `owner`, and slot identifiers must serialize as
  stable strings
- absent optional data must serialize as `null` or be omitted consistently; the
  project should choose one policy and keep it fixed
- summary counters and `complete`/`truncated` state must be present so callers
  can distinguish a clean run from a truncated one
- JSON must not require parsing human `message` text to understand the result

Recommended top-level structure:

```json
{
  "ok": false,
  "complete": true,
  "mode": "selected_state",
  "selected_slot": "slot_a",
  "selected_generation": 42,
  "selected_txn_id": 42,
  "page_size": 4096,
  "page_count": 120,
  "summary": {
    "error_count": 1,
    "warning_count": 0,
    "info_count": 2,
    "unreachable_scan_enabled": false,
    "truncated": false
  },
  "slot_a": { "...": "..." },
  "slot_b": { "...": "..." },
  "issues": [
    {
      "severity": "error",
      "kind": "freelist_reachable_page_overlap",
      "scope": "selected_state",
      "metadata_slot": "slot_a",
      "page_id": 42,
      "page_type": "leaf",
      "owner": "freelist_reusable",
      "related_page_id": 17,
      "related_owner": "btree_leaf",
      "generation": 42,
      "txn_id": 42,
      "pending_free_txn_id": null,
      "message": "page 42 is both reachable and listed as reusable"
    }
  ]
}
```

Stability rules:

- JSON ordering should be deterministic even if callers should not rely on it
- messages may evolve, but examples and tests must assert on structured fields
- JSON output must omit user payload bytes by default

## 9. Traversal Scope

M7 must keep the selected-state trust boundary explicit.

### 9.1 Common Start

Every verify mode must:

1. validate the header
2. validate slot A and slot B independently
3. re-run normal metadata selection
4. stop selected-state traversal if no slot can be selected

### 9.2 `metadata_only`

This mode may report:

- header failures
- incompatible format or required-feature failures
- slot-local metadata invalidity
- equal-generation conflicts
- `txn_id` regression across otherwise valid slots

This mode must not:

- traverse the selected tree
- walk overflow chains
- decode the full freelist image beyond critical slot validation needed to
  accept or reject the referenced metadata state

### 9.3 `selected_state`

This mode must additionally:

- validate selected root and selected freelist pages as critical state
- traverse reachable branch and leaf pages
- validate B+Tree ordering, child bounds, slot offsets, and payload bounds
- validate reachable overflow chains through the same M6 helper logic used by
  normal reads
- validate selected freelist pages and decoded `reusable` and `pending_free`
  entries through the same M6 helper logic used by `open()`
- track ownership across tree pages, overflow pages, freelist pages,
  `reusable`, and `pending_free`

### 9.4 `selected_state_with_unreachable_scan`

This mode must additionally:

- scan unclaimed physical pages with `page_id < selected page_count`
- report them as `warning` or `info` only; they remain non-authoritative
- classify pages with `page_id >= selected page_count` as `ignored_tail_page`
  informational findings when they are physically present

Scope rules:

- unreachable pages below selected `page_count` must never be treated as live
  or reusable by verify
- ignored tail pages must remain informational because M0 allows them after
  failed or partial commits
- verify must not invent a second recovery path by trusting unreachable pages
  more than the selected metadata snapshot

## 10. Tool Exit Rules

`zi6db verify <path>` must have simple and deterministic exit behavior.

Exit expectations:

- exit `0`: setup succeeded and the report contains no `error` issues
- non-zero exit for reportable `error` issues
- non-zero exit for fatal setup failures where no trustworthy report can be
  produced, such as unreadable file, unsupported format handshake before a
  report can be built, or lock acquisition failure

Tool rules:

- `warning` and `info` findings alone do not make the CLI fail
- `lock_conflict` must not masquerade as generic corruption in either prose or
  JSON
- default human output should include path, selected slot, summary counts,
  whether unreachable scanning was enabled, and a bounded list of issues
- JSON mode and human mode must agree on counts and success/failure outcome

## 11. Implementation Steps

### 11.1 Freeze Public Types

- define `VerifyOptions`, `VerifyMode`, `VerifyReport`, `VerifySummary`,
  `VerifySlotResult`, `VerifyIssue`, `VerifySeverity`, and `VerifyIssueKind`
- keep names aligned with M4 and M6 internal checker types to minimize mapping

### 11.2 Reuse M4 And M6 Validators

- adapt M4 slot-check and selected-state report builders into public verify
  report components
- reuse M6 overflow and freelist validators for public issue generation instead
  of introducing parallel decoding logic
- reuse M6 ownership walkers so duplicate ownership reporting remains identical

### 11.3 Add Public Serialization

- implement stable JSON serialization for the report and issues
- ensure string forms for kinds, severity, scope, owner, and slot identifiers
  are documented and tested
- keep CLI human rendering separate from JSON serialization

### 11.4 Add Tool Surface

- expose `zi6db verify <path>`
- support traversal mode selection and JSON output
- map fatal setup failures and report-level failures to documented exit status

## 12. Test Plan

### 12.1 VerifyOptions And Traversal Tests

- `metadata_only` reports slot and selection failures without tree traversal
- `selected_state` reports reachable corruption while skipping unreachable-page
  findings
- `selected_state_with_unreachable_scan` reports unreachable pages within
  selected `page_count` only in that mode
- `include_informational = false` suppresses `info` issue records without
  changing summary truth about scan mode or truncation
- `max_issues` truncates deterministically and emits `incomplete_report`

### 12.2 Issue And Severity Tests

- inactive-slot checksum mismatch remains non-fatal and is not promoted to
  selected-state corruption when the other slot wins
- equal valid generations produce `metadata_equal_generation` with `error`
  severity
- selected root or freelist corruption produces `error`
- unreachable page within selected `page_count` in deep mode produces
  non-fatal `warning` or documented `info`
- ignored tail pages beyond selected `page_count` produce `info`
- lock conflict produces `lock_conflict` and non-zero tool exit

### 12.3 JSON Stability Tests

- clean and corrupt reports serialize with the documented top-level fields
- issue kinds and severity strings match the documented spellings exactly
- optional fields follow the chosen `null` versus omission policy consistently
- CLI JSON mode and API serialization agree on counts and truncation state
- messages can vary without breaking structured-field assertions
- JSON output omits user payload bytes by default

### 12.4 Exit Rule Tests

- clean database returns exit `0`
- database with only warnings returns exit `0`
- database with one or more `error` findings returns non-zero
- unreadable path returns non-zero fatal setup failure
- lock conflict returns non-zero and is classified separately from corruption

### 12.5 Cross-Feature Regression Tests

- verify uses the same overflow corruption detection as M6 live reads
- verify uses the same freelist validation and duplicate ownership detection as
  M6 selected-state helpers
- verify preserves M4 slot result semantics for both valid and invalid
  inactive slots

## 13. Acceptance

This feature is complete for M7 when:

- `verify()` is public, read-only, and path-based
- `VerifyOptions` defines stable traversal modes, informational filtering, and
  bounded issue collection
- verify reports preserve per-slot results, selected-state outcome, and
  structured issue records
- stable public issue kinds cover metadata, B+Tree, overflow, freelist,
  unreachable-page, truncation, and lock-conflict cases
- the public severity model is fixed to `error`, `warning`, and `info`
- JSON output is documented, machine-readable, and stable enough for downstream
  tooling before `v1.0.0`
- verify explicitly states whether unreachable-page scanning ran
- unreachable pages inside selected `page_count` remain opt-in and
  non-authoritative
- ignored tail pages remain informational and do not fail the tool
- CLI exit status depends on fatal setup failure or `error` findings only
- tests cover option behavior, issue classification, JSON stability, traversal
  scope, truncation, and exit rules
- the implementation reuses M4 and M6 validation seams instead of introducing a
  second checker or decoder stack
