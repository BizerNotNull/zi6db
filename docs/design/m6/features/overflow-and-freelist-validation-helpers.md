# M6 Feature Plan: Overflow And Freelist Validation Helpers

## Goal

M6 adds reusable read-only validation helpers for overflow chains and freelist
images so large-value storage and page reuse can be checked with the same rules
used by normal reads, `open()`, crash tests, and later M7 verify tooling.

This feature exists to keep M6 correctness work from splitting into multiple
inconsistent validators. The helpers should let M6 detect selected-state
corruption precisely, explain duplicate ownership clearly, and hand M7 a stable
checker seam for verify and compaction preflight without changing M0 storage
semantics.

## Scope

In scope:

- Validate selected-state freelist pages and decoded freelist entries.
- Validate live overflow references and reachable overflow chains.
- Distinguish `open()` critical-path checks from deeper checker-only traversal.
- Detect duplicate ownership across tree pages, overflow pages, freelist pages,
  `reusable`, and `pending_free`.
- Return structured findings with stable codes, severity, scope, and owner
  context.
- Support M6 tests for corruption, reuse safety, and handoff readiness.
- Expose internal walkers and result shapes that M7 verify and optional
  compaction can reuse directly.

Out of scope:

- A public `verify()` API or CLI. That remains M7 scope.
- Repair, salvage, truncation, compaction, or free-space inference from
  unreachable pages.
- Full physical-file scanning during normal `open()`.
- Cross-process reader tracking or lock findings. Those remain M7 scope.
- Any change to page layout, metadata selection, checksum rules, or the M0
  commit protocol.

## Design Principles

- Validation must be read-only and must never mutate freelist state, promote
  pending pages, or rewrite overflow references.
- `open()` must validate only what M0 and M6 require for a trustworthy selected
  snapshot: selected metadata, selected freelist structure, and critical page
  sanity.
- Deep traversal must be opt-in so normal startup cost stays bounded.
- Normal reads of live overflow values must still validate the specific chain
  they dereference, even when a full deep check is not running.
- Findings must use stable machine-readable codes rather than prose-only
  assertions.
- Duplicate ownership must identify both sides of the conflict instead of
  collapsing all overlap into generic corruption.
- The same validators should serve M6 read paths, checker paths, corruption
  tests, and M7 verify walkers.

## Validation Scope

M6 should split validation into three layers.

### Layer 1: `open()` Critical Path

`open()` must validate only the selected metadata snapshot and the selected
freelist image strongly enough to trust future allocation and recovery
decisions.

Required checks:

- `freelist_page_id` is in range for the selected metadata.
- every selected freelist page has the expected page type and a valid checksum
- freelist chain sequencing and next-page pointers are internally consistent
- decoded freelist entries never reference pages `0`, `1`, or `2`
- decoded freelist entries are `< selected_metadata.page_count`
- no page appears twice inside `reusable`
- no page appears twice across `pending_free` groups
- no page appears in both `reusable` and `pending_free`
- selected freelist pages do not name the new freelist image itself as free
- decoded freelist state is canonical enough that allocation will not return the
  same page twice

`open()` is not required to traverse every bucket root, leaf overflow
reference, or unreachable page below `page_count`. It should fail with
`Corruption` when the selected freelist itself is malformed, because that state
is directly trusted for M6 allocation and recovery.

### Layer 2: Live Read Validation

Whenever a read path follows an overflow reference from a selected live leaf, it
must validate that specific chain before returning bytes.

Required checks:

- referenced first page id is in range
- referenced pages decode as `PAGE_TYPE_OVERFLOW`
- chain length, monotonic index, and next pointers match the leaf reference
- total payload bytes equal `logical_value_length`
- no page repeats within the same chain
- every chain page stays within selected `page_count`

Malformed live overflow chains must fail the read with `Corruption`. M6 must
not return partial value bytes or guess a shorter chain.

### Layer 3: Deep Check

An internal deep checker should validate selected-state ownership across the
tree, overflow chains, freelist pages, `reusable`, and `pending_free`.

Deep check responsibilities:

- traverse selected bucket and B+Tree reachability
- enumerate every live overflow chain exactly once per owning leaf reference
- enumerate every selected freelist page in the freelist chain
- check ownership overlap between reachable tree pages, overflow chain pages,
  freelist pages, reusable entries, and pending entries
- optionally classify ignored tail pages outside selected `page_count` as
  informational without promoting or reclaiming them

Deep check must remain non-authoritative for file repair. It explains selected
state and suspicious leftovers, but it must not redefine recovery.

## Open Vs Deep-Check Boundary

M6 should freeze the boundary so runtime behavior and tests stay predictable.

`open()` must do:

- header and metadata selection through the existing M4 rules
- selected root and selected freelist critical-page validation
- freelist decoding and duplicate/overlap validation needed for safe reuse

`open()` must not do:

- a full scan of all branch and leaf pages solely to prove no page is leaked
- a full traversal of every live overflow value in the database
- an unreachable-page scan within selected `page_count`
- any interpretation of pages beyond selected `page_count` beyond optional
  informational reporting in checker-only modes

Deep check may do:

- full selected-state tree traversal
- full live overflow enumeration
- ownership overlap reporting across all selected-state classes
- optional ignored-tail and unreachable-page classification for diagnostics

Rationale:

- freelist corruption is startup-critical because M6 recovery and allocation
  trust it immediately
- overflow corruption becomes critical when a read or delete path dereferences a
  specific live chain
- whole-file ownership proof is valuable, but it is an internal checker and M7
  verify concern rather than a mandatory startup cost

## Recommended API Shape

M6 may keep these helpers internal, but the implementation should converge on a
shape close to:

```zig
pub fn validateFreelistImage(
    file: *StorageFile,
    metadata: MetadataSnapshot,
    options: FreelistValidationOptions,
) !FreelistValidationReport;

pub fn validateOverflowChain(
    file: *StorageFile,
    metadata: MetadataSnapshot,
    reference: OverflowReference,
    options: OverflowValidationOptions,
) !OverflowValidationReport;

pub fn checkSelectedOwnership(
    db: *DB,
    options: OwnershipCheckOptions,
) !OwnershipCheckReport;
```

Rules:

- the freelist validator must be usable from `open()` before the database is
  fully published
- the overflow validator must be usable from normal value reads, delete/update
  paths, and the internal checker
- the ownership checker must reuse tree, freelist, and overflow walkers instead
  of re-decoding them with separate logic

## Structured Findings

The M6 finding model should extend earlier checker work without renaming the
existing classification approach.

```zig
const CheckFinding = struct {
    severity: CheckSeverity,
    code: CheckCode,
    scope: FindingScope,
    owner: ?FindingOwner = null,
    conflicting_owner: ?FindingOwner = null,
    page_id: ?u64 = null,
    related_page_id: ?u64 = null,
    metadata_slot: ?MetadataSlotId = null,
    generation: ?u64 = null,
    txn_id: ?u64 = null,
    pending_free_txn_id: ?u64 = null,
    message: []const u8,
};

const FindingScope = enum {
    selected_freelist,
    live_overflow_read,
    selected_ownership,
    unreachable_scan,
};

const FindingOwner = enum {
    metadata,
    freelist_page,
    freelist_reusable,
    freelist_pending,
    btree_branch,
    btree_leaf,
    overflow_chain,
    physical_tail,
};
```

Recommended new `CheckCode` values:

- `freelist_page_checksum_mismatch`
- `freelist_page_sequence_invalid`
- `freelist_reserved_page`
- `freelist_page_out_of_range`
- `freelist_duplicate_reusable_page`
- `freelist_duplicate_pending_page`
- `freelist_reusable_pending_overlap`
- `freelist_self_reference`
- `overflow_reference_out_of_range`
- `overflow_page_checksum_mismatch`
- `overflow_page_wrong_type`
- `overflow_chain_cycle`
- `overflow_chain_length_mismatch`
- `overflow_chain_payload_length_mismatch`
- `overflow_chain_page_out_of_range`
- `overflow_chain_duplicate_page`
- `duplicate_page_ownership`
- `ignored_tail_page`
- `unreachable_page_within_page_count`
- `incomplete_check`

Severity guidance:

- `corruption` for malformed selected freelist state, malformed live overflow
  chains, or duplicate ownership inside selected state
- `warning` only for non-authoritative deeper-scan findings that do not change
  the selected-state outcome
- `info` for ignored tail pages outside selected `page_count`

M6 tests should assert on codes and owner fields rather than matching message
text.

## Duplicate Ownership Detection

M6 should treat duplicate ownership as a first-class validation concern, not a
derived side effect of other checks.

### Ownership Classes

During deep traversal, each selected-state page should belong to at most one of:

- selected B+Tree structure as branch or leaf
- selected live overflow chain
- selected freelist page chain
- freelist `reusable`
- freelist `pending_free[freeing_txn_id]`

The checker should track both the first claim and any later conflicting claim.

### Detection Rules

Required duplicate checks:

- same page appears twice in one overflow chain
- same page appears in two different live overflow chains
- same page appears twice in freelist `reusable`
- same page appears in two different `pending_free` groups
- same page appears in both `reusable` and `pending_free`
- same page is reachable from the selected tree and also appears in freelist
  free space
- same page is a selected freelist page and also appears in freelist contents
- same page is used by both a B+Tree page and a live overflow chain

### Reporting Rules

Duplicate ownership findings should include:

- conflicting page id
- primary owner class
- conflicting owner class
- related owner page id when known, such as the leaf page referencing an
  overflow chain or the freelist page that encoded an entry
- `pending_free_txn_id` when the conflicting owner is a pending group

This separation lets M7 verify report more actionable diagnostics such as
"reachable page also listed in reusable" versus "overflow page shared by two
live values" without reclassifying M6 findings later.

## Test Plan

M6 should add focused tests around helper behavior in addition to the broader
overflow and freelist feature tests.

### Freelist Validation Tests

- selected freelist page checksum mismatch fails `open()` and checker with the
  freelist checksum code
- selected freelist references reserved page `0`, `1`, or `2`
- selected freelist references page id `>= page_count`
- selected freelist contains a duplicate reusable page
- selected freelist contains a duplicate pending page across groups
- selected freelist contains overlap between reusable and pending
- selected freelist references one of its own live freelist pages as free

### Overflow Validation Tests

- read of live overflow value detects wrong page type
- read of live overflow value detects checksum mismatch
- read of live overflow value detects cycle
- read of live overflow value detects shorter-than-declared chain
- read of live overflow value detects longer-than-declared chain
- delete or replace of a corrupt live overflow value fails without guessing
  pages to free

### Duplicate Ownership Tests

- deep checker reports overflow page shared by two live values
- deep checker reports page reachable from tree and also listed in reusable
- deep checker reports selected freelist page also listed in pending
- deep checker reports duplicate pending ownership across freeing transaction
  groups

### Boundary Tests

- `open()` succeeds on a database with valid selected metadata and freelist even
  when an unrelated unreachable page inside selected `page_count` exists, unless
  deep scan is explicitly requested
- deep checker reports ignored tail pages beyond selected `page_count` as
  `info`, not `corruption`
- normal read of an inline value does not invoke overflow validation
- normal read of an overflow value validates only the referenced chain, not all
  chains in the file

## M7 Verify And Compaction Handoff

M6 must leave a clear seam that M7 can adopt without changing any M6 storage
rules.

Verify handoff:

- provide read-only walkers for selected freelist pages, freelist entries, live
  overflow chains, and ownership claims
- preserve stable finding codes, scopes, and owner classes
- classify ignored tail pages outside selected `page_count` as informational
- keep selected-state corruption separate from optional deeper-scan findings
- allow M7 verify to aggregate multiple findings without reimplementing page
  decoders

Compaction handoff:

- expose ownership classification that separates live tree pages, live overflow
  pages, freelist pages, reusable entries, pending entries, and ignored tail
  pages
- keep free-page accounting and duplicate detection independent from current
  physical layout so M7 compaction can remap live pages cleanly
- avoid any M6 helper behavior that implicitly truncates, relocates, or
  canonicalizes physical tail bytes

M7 should be able to consume the M6 validators as authoritative decoders while
adding public reporting, optional physical-page scans, and page-remapping logic
on top.

## Acceptance

This feature is complete for M6 when:

- selected freelist validation is strong enough for `open()` to reject malformed
  free-space state before allocation can trust it
- live overflow reads validate the referenced chain and return `Corruption`
  rather than partial bytes on malformed chains
- the open-vs-deep-check boundary is explicit in docs, tests, and helper APIs
- duplicate ownership is detected separately for overflow sharing, freelist
  duplication, freelist overlap, and live-page-versus-free-page conflicts
- structured findings include stable codes, severity, scope, and owner context
- tests cover freelist corruption, overflow corruption, duplicate ownership, and
  informational ignored-tail reporting
- deep checker results are reusable by M7 verify without re-decoding overflow or
  freelist structures through separate logic
- the helpers remain read-only and do not promote pending pages, repair
  corruption, infer reusable space from unreachable pages, or change M0/M6
  storage semantics
