# M2 Feature Plan: Structural Checker

## Goal

The M2 structural checker is an internal read-only validation path for the
single ordered key space introduced in M2. It should prove that the selected
B+Tree root is structurally usable for point lookup after clean writes, clean
reopen, split propagation, and no-rebalance deletes.

The checker is also the seed for M7 verify tooling. M2 should keep it small and
focused on reachable B+Tree pages, while returning enough structured detail that
later milestones can extend it without replacing the result shape.

## Scope

In scope:

- Validate the selected metadata root page id and root page type.
- Traverse reachable `PAGE_TYPE_LEAF` and `PAGE_TYPE_BRANCH` pages.
- Validate page checksum, encoded page id, page type, slot bounds, key ordering,
  child pointer bounds, child type, child ranges, duplicate child references,
  duplicate reachable pages, and root sanity.
- Accept underfull and empty pages produced by M2 deletes when routing remains
  correct.
- Return a structured `CheckReport` used by tests and future verify tooling.
- Support targeted root-only validation and full tree validation.

Out of scope:

- Full header and dual-metadata recovery checking. M4 expands this area.
- Complete freelist reachability and reusable-page accounting.
- Overflow chain validation.
- Bucket catalog validation.
- Crash-outcome proof or repair.
- Minimum page occupancy enforcement after delete.

## Public Shape

M2 may keep the checker behind an internal API, but the call shape should be
close to:

```zig
pub fn check(db: *DB, options: CheckOptions) !CheckReport;
```

`check` is allowed on read-only database handles. It must not acquire the writer
guard, mutate metadata, allocate pages, repair pages, or scan pages that are not
reachable from the metadata snapshot being checked.

## CheckOptions

Recommended fields:

```zig
const CheckOptions = struct {
    mode: CheckMode = .full_tree,
    metadata: ?MetadataSnapshot = null,
    max_findings: ?usize = null,
};

const CheckMode = enum {
    root_only,
    full_tree,
};
```

Rules:

- `metadata = null` means the currently selected in-memory metadata snapshot.
- `root_only` validates the selected root page id, checksum, encoded page id,
  and root page type, but does not recursively descend.
- `full_tree` validates every reachable branch and leaf page.
- `max_findings` may stop traversal early after enough findings are collected,
  but the report must mark itself incomplete.

## CheckReport

`CheckReport` should be structured rather than a boolean so tests can assert
specific corruption classes and M7 can reuse the shape.

Recommended fields:

```zig
const CheckReport = struct {
    ok: bool,
    complete: bool,
    mode: CheckMode,
    checked_root_page_id: PageId,
    page_count: PageId,
    tree_height: ?u32,
    reachable_tree_pages: u64,
    reachable_leaf_pages: u64,
    reachable_branch_pages: u64,
    max_depth: ?u32,
    min_depth: ?u32,
    findings: []CheckFinding,
};
```

`ok` is true only when there are no findings with severity `corruption` or
`incompatible`. Informational findings may still be present.

`tree_height` is the leaf depth plus one when all reachable leaves are at the
same depth. If traversal is incomplete or leaf depths differ, it should be null.

## Findings

Each finding should include a stable code. Do not rely on prose text for tests.

```zig
const CheckFinding = struct {
    severity: CheckSeverity,
    code: CheckCode,
    page_id: ?PageId = null,
    parent_page_id: ?PageId = null,
    child_page_id: ?PageId = null,
    depth: ?u32 = null,
    key_index: ?u16 = null,
    message: []const u8,
};

const CheckSeverity = enum {
    ok,
    info,
    warning,
    corruption,
    incompatible,
};
```

M2 should define at least these `CheckCode` values:

- `root_page_out_of_range`
- `invalid_root_page_type`
- `page_read_failed`
- `page_checksum_mismatch`
- `page_id_mismatch`
- `invalid_page_type`
- `invalid_slot_directory`
- `payload_out_of_bounds`
- `payload_overlap`
- `unsupported_leaf_value_flags`
- `leaf_keys_unsorted`
- `duplicate_leaf_key`
- `branch_keys_unsorted`
- `branch_child_out_of_range`
- `branch_child_self_reference`
- `branch_duplicate_child_pointer`
- `branch_invalid_child_type`
- `duplicate_reachable_page`
- `child_range_violation`
- `leaf_depth_mismatch`
- `invalid_empty_branch_root`
- `accepted_underfull_page`
- `accepted_empty_non_root_leaf`
- `accepted_empty_branch_root_tree`
- `incomplete_check`

Accepted M2 delete shapes should be reported as `info` only when useful for
tests or diagnostics. They must not make `ok` false.

## Validation State

Traversal should carry a range frame instead of only visiting pages:

```zig
const Bound = struct {
    key: ?[]const u8,
    inclusive: bool,
};

const KeyRange = struct {
    lower: Bound,
    upper: Bound,
};

const VisitFrame = struct {
    page_id: PageId,
    parent_page_id: ?PageId,
    depth: u32,
    range: KeyRange,
    is_root: bool,
};
```

Use the same unsigned-byte lexicographic comparator as search and mutation.
Checker-only comparison behavior must not drift from production routing.

## Root Validation

Root validation runs before full traversal:

1. Confirm `root_page_id >= FIRST_DATA_PAGE_ID`.
2. Confirm `root_page_id < metadata.page_count`.
3. Read the root page by physical page id.
4. Validate checksum before decoding the body.
5. Confirm the encoded page id equals the physical page id.
6. Confirm root page type is `PAGE_TYPE_LEAF` or `PAGE_TYPE_BRANCH`.

Root-specific rules:

- A root leaf may be empty.
- A root branch must have at least one separator and a valid rightmost child.
- A root branch that represents an empty logical tree after deletes is accepted
  only if all descendants satisfy their ranges and all leaves are empty.
- A root may not be a freelist, metadata, header, overflow, unknown, or reserved
  page type.

## Leaf Validation

For each reachable leaf:

1. Confirm page type is `PAGE_TYPE_LEAF`.
2. Confirm entry count and slot directory fit inside the page.
3. Confirm each key and value payload range stays inside the page.
4. Confirm payload ranges do not overlap unless the codec explicitly supports
   sharing. M2 should not use shared payloads.
5. Confirm only M2-supported inline value flags are present.
6. Confirm keys are strictly sorted by the shared comparator.
7. Confirm there are no duplicate keys.
8. Confirm every key is inside the `KeyRange` inherited from branch traversal.
9. Record leaf depth for same-depth validation.

Underfull rules:

- A non-root leaf with fewer entries than a balanced B+Tree would normally
  require is valid in M2.
- An empty non-root leaf is valid after deletes if its inherited range is still
  internally consistent and no remaining key should route into a different
  subtree.
- Empty accepted leaves may produce `accepted_empty_non_root_leaf` info
  findings.

## Branch Validation

For each reachable branch:

1. Confirm page type is `PAGE_TYPE_BRANCH`.
2. Confirm key count and slot directory fit inside the page.
3. Confirm every separator payload range stays inside the page.
4. Confirm separator keys are strictly sorted.
5. Confirm child page ids are `>= FIRST_DATA_PAGE_ID` and `< page_count`.
6. Confirm no child page id equals the branch page id.
7. Confirm child page ids are unique within the branch.
8. Confirm no child page id has already been visited in the reachable tree.
9. Confirm children decode as `PAGE_TYPE_LEAF` or `PAGE_TYPE_BRANCH`.
10. Derive child ranges using the M2 routing rule and recurse.

The checker must validate branch routing by ranges, not by requiring each
separator to equal the current first key in the right child. M2 deletes may leave
stale separators, and those stale separators are valid when they still route all
possible point lookups to the subtree whose range contains the lookup.

## Range Validation

M2 search descends by finding the first separator greater than the search key. If
one exists, search follows that separator's left child. Otherwise, search follows
the rightmost child.

For sorted separators `s0, s1, ..., sn-1`, the checker should derive child
ranges as:

- child `0`: parent lower bound through upper bound `< s0`
- child `i`: lower bound `>= s(i-1)` through upper bound `< si`
- rightmost child: lower bound `>= s(n-1)` through parent upper bound

This model treats each separator as the lower bound for the subtree to its
right. A stale separator remains valid if all descendant leaf keys stay within
the derived range and all search keys in that range route to that child.

Range validation should reject:

- a leaf key lower than its inclusive lower bound
- a leaf key greater than or equal to its exclusive upper bound
- a branch separator outside the branch's inherited range
- child ranges where lower bound is not strictly less than upper bound unless the
  child is an accepted empty subtree created by delete
- any branch shape that makes an existing key unreachable by normal point lookup

## Reachability

Full-tree traversal must maintain a visited set of reachable tree page ids.

Rules:

- Visiting the same tree page twice is corruption.
- A branch pointing to itself is corruption even before the duplicate visit is
  observed.
- Duplicate child pointers within one branch are corruption.
- Reachable pages must be inside metadata `page_count`.
- The checker should not scan unreachable tail pages in M2.
- Unreachable non-critical pages are not M2 checker corruption.

M4 and M7 may add broader file-level reporting for ignored unreachable pages.
M2 should keep reachability scoped to the selected tree.

## Accepted Underfull Deletes

M2 deliberately does not merge, rebalance, redistribute, or collapse branch roots
after delete. The checker must encode that product decision explicitly.

Accepted:

- Underfull non-root leaves.
- Empty non-root leaves.
- Underfull non-root branches if all child pointers are valid and ranges remain
  correct.
- A root leaf with zero entries.
- A branch root left behind after all logical entries were deleted, when every
  reachable leaf is empty and branch ranges are internally consistent.
- Stale separator keys after delete, when they remain safe routing lower bounds.

Rejected:

- Stale separators that route a remaining key to the wrong child.
- Any remaining key outside its inherited child range.
- Empty or underfull pages with malformed slots, bad checksums, wrong encoded
  page ids, unsupported page types, or duplicate reachability.
- A root branch with no separators or without a valid rightmost child.
- A branch-root empty-tree representation where some reachable leaf still has a
  key outside the root's derived ranges.

## Corruption Tests

M2 should include corruption tests that mutate a clean test database at the page
bytes or codec fixture level. Test names should describe the behavior being
detected.

Required checker failure cases:

- Root page id below `FIRST_DATA_PAGE_ID`.
- Root page id greater than or equal to metadata `page_count`.
- Root page with invalid page type.
- Reachable page checksum mismatch.
- Reachable page encoded id mismatch.
- Leaf slot directory outside the page.
- Leaf key payload outside the page.
- Leaf value payload outside the page.
- Overlapping leaf payloads.
- Unsupported leaf value flags.
- Unsorted leaf keys.
- Duplicate leaf keys.
- Branch slot directory outside the page.
- Unsorted branch separators.
- Branch child pointer below `FIRST_DATA_PAGE_ID`.
- Branch child pointer greater than or equal to metadata `page_count`.
- Branch child pointer to itself.
- Duplicate child pointer in one branch.
- Duplicate reachable page through different branches.
- Branch child points at a freelist or overflow page.
- Leaf key outside inherited child range.
- Branch separator outside inherited parent range.
- Leaves at mismatched depths.
- Malformed branch-root empty-tree representation.

Required accepted-state cases:

- Clean empty root leaf passes.
- Root leaf with inserts passes.
- Root split with two leaves passes.
- Multi-level tree after branch split passes.
- Delete first key in a right subtree passes when lookup correctness is
  preserved.
- Delete all keys from a non-root leaf leaves an accepted empty leaf.
- Delete many keys after splits leaves accepted underfull pages.
- Delete all keys from a multi-level tree passes if the implementation keeps a
  branch-root empty-tree representation.

Each corruption test should assert both the public error mapping where applicable
and the specific `CheckCode` returned by `CheckReport`.

## Implementation Steps

1. Define `CheckOptions`, `CheckReport`, `CheckFinding`, `CheckSeverity`, and
   `CheckCode` in the tree checker module.
2. Add shared helpers for collecting bounded findings and marking incomplete
   reports.
3. Implement root-only validation using existing page read, checksum, and common
   page decode helpers.
4. Implement leaf structural validation on decoded leaf pages.
5. Implement branch structural validation on decoded branch pages.
6. Add the traversal stack or bounded recursion with `VisitFrame` and inherited
   `KeyRange`.
7. Add a visited set for reachable tree page ids.
8. Add same-depth leaf tracking.
9. Add accepted-underfull and accepted-empty diagnostics as `info` findings.
10. Wire `DB.check` or the internal equivalent to selected metadata snapshots.
11. Use the checker in M2 split, delete, persistence, randomized, and corruption
    tests.
12. Keep all checker helpers read-only and independent from mutation planning.

## M7 Verify Handoff

M2 should leave these extension seams for M7:

- Stable finding severities and classification codes.
- Page id, parent page id, child page id, depth, and key index fields in
  findings.
- A report shape that can add header, metadata-slot, freelist, overflow, bucket,
  lock, and file-level findings later.
- A traversal implementation that can be reused by a user-facing verify command.
- Clear separation between selected-tree corruption and informational
  unreachable-page diagnostics.
- No repair behavior and no mutation through the checker path.

M7 can promote the internal checker into a public verify feature, add richer
report formatting, scan non-critical unreachable pages, validate full freelist
ownership, and include bucket/cursor/overflow invariants without changing the M2
B+Tree range rules.

## Acceptance

The structural checker feature is complete for M2 when:

- `CheckReport` returns structured findings with stable severity and code
  fields.
- Root-only mode validates root range, checksum, encoded page id, and root type.
- Full-tree mode traverses every reachable branch and leaf from the selected
  metadata root.
- Reachable page checksum, encoded page id, page type, slot bounds, and payload
  bounds are validated before page contents are trusted.
- Leaf key ordering, duplicate keys, value flags, and inherited ranges are
  validated.
- Branch separator ordering, child pointer ranges, duplicate child references,
  child page types, and self references are validated.
- Reachability detects duplicate tree visits and does not scan unrelated pages.
- Range validation follows the M2 branch routing rule and allows safe stale
  separators after delete.
- All reachable leaves have the same depth unless a deliberately documented M2
  exception is added.
- Underfull and empty pages produced by no-rebalance deletes are accepted only
  when lookup routing remains correct.
- A branch-root empty-tree representation is accepted only when all descendants
  are empty and ranges are internally consistent.
- Corruption tests cover malformed root, leaf, branch, reachability, checksum,
  and child-range cases.
- Accepted-state tests cover clean trees, split trees, underfull deletes, empty
  non-root leaves, and all-keys-deleted shapes.
- The checker is used by representative M2 persistence and randomized tests.
- The implementation remains read-only and leaves a clear handoff path for M7
  verify tooling.
