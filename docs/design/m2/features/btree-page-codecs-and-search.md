# M2 Feature Plan: B+Tree Page Codecs And Search

This feature implements the read-only B+Tree foundation for M2: the shared byte
comparator, leaf and branch page codecs, strict slot/payload validation, branch
routing, and point-search traversal. Mutation, copy-on-write publication, and
split propagation build on this work but are not part of this feature except
where codec decisions must leave the correct seams.

## 1. Scope

### 1.1 In Scope

- One shared raw-byte comparator used by codecs, search, mutation, and checker.
- Decoding and encoding M0 leaf page bodies.
- Decoding and encoding M0 branch page bodies.
- In-page binary search for leaf entries and branch separators.
- Validation of slot directories, payload ranges, flags, ordering, and child
  pointers while decoding.
- Point-search traversal from the selected metadata root through branch and leaf
  pages.
- Structured search results that later insert, replace, delete, and checker code
  can reuse.
- Unit and fixture tests for comparator behavior, codec round trips, malformed
  pages, branch routing, and read-only search.
- Checker handoff data and validation responsibilities.

### 1.2 Out Of Scope

- Public KV API finalization beyond any narrow calls needed to exercise search.
- Insert, replace, delete, split, root split, and metadata publication.
- Freelist reuse or pending-free updates.
- Overflow value decoding beyond rejecting unsupported leaf value flags.
- Cursor, range scan, prefix scan, or backward iteration semantics.
- Crash-fault or power-loss testing.

## 2. Format Dependencies

This feature must follow the frozen M0 storage format:

- Page `0` is the immutable file header.
- Page `1` and page `2` are metadata slots.
- Page ids `>= 3` are normal data pages.
- All integer fields are little-endian.
- Normal pages use the common page header with `page_id`, `page_type`,
  `page_generation`, flags, and checksum.
- Branch pages contain `key_count`, `rightmost_child`, a slot directory, and
  variable-length separator key payloads.
- Leaf pages contain `entry_count`, a reserved right-sibling field, a slot
  directory, and variable-length key/value payloads.
- The checksum must validate before a page body is interpreted.
- `OVERFLOW` is only format-reserved in M2; unsupported overflow value flags must
  be rejected cleanly.

## 3. Implementation Sequence

1. Add the comparator and exhaustive comparator tests.
2. Add shared page-body bounds helpers for slot directories and payload ranges.
3. Implement leaf decode validation and borrowed entry views.
4. Implement leaf encode and round-trip tests.
5. Implement branch decode validation and borrowed separator/child views.
6. Implement branch encode and round-trip tests.
7. Add leaf and branch in-page binary search helpers.
8. Implement branch child routing using the M2 lower-bound separator rule.
9. Implement root-to-leaf point search over validated pages.
10. Add checker handoff structures and malformed-page test fixtures.

## 4. Comparator

Create one comparator module, suggested as `src/tree/comparator.zig`.

Required behavior:

- Compare keys as raw byte slices.
- Treat every byte as unsigned `u8`.
- Return the first differing byte comparison.
- If all common bytes are equal, sort the shorter key first.
- Equal length and equal bytes are the same key.

Required API shape:

```zig
pub const Ordering = enum { lt, eq, gt };

pub fn compare(a: []const u8, b: []const u8) Ordering;
pub fn lessThan(_: void, a: []const u8, b: []const u8) bool;
```

The comparator must not use locale, UTF-8, signed-byte ordering, or host-endian
integer reinterpretation. All B+Tree code, including tests and checker logic,
must import this module instead of duplicating comparison code.

Comparator tests must cover:

- empty key versus empty key
- empty key versus non-empty key
- prefix keys such as `a`, `aa`, and `ab`
- embedded `0x00`
- high-bit bytes such as `0x80` and `0xff`
- equal binary keys from distinct buffers
- sorted fixtures that mix short, long, and binary keys

## 5. Leaf Page Codec

### 5.1 Decoded Representation

Use a borrowed representation so read-only search does not copy key/value bytes:

```zig
pub const LeafEntry = struct {
    key: []const u8,
    value: []const u8,
    flags: LeafValueFlags,
};

pub const DecodedLeaf = struct {
    page_id: PageId,
    generation: u64,
    right_sibling: ?PageId,
    entries: []const LeafEntry,
};
```

The exact Zig field names may follow M1 conventions, but the ownership contract
must be explicit: decoded entries borrow from the page buffer and cannot outlive
that buffer.

### 5.2 Decode Validation

Leaf decoding must fail with `Corruption` or an internal codec error that maps to
`Corruption` when any invariant is violated:

- Common header checksum is invalid.
- Common header page type is not `PAGE_TYPE_LEAF`.
- Encoded `page_id` does not match the physical page id requested by the caller.
- `entry_count` cannot fit in the page after accounting for fixed body fields and
  slot size.
- Slot directory extends outside the page.
- Any key or value range starts before the payload area, extends past the page,
  or overflows integer arithmetic.
- Any key or value range overlaps the common header, leaf fixed fields, or slot
  directory.
- Payload ranges overlap each other. M2 should not use shared payload storage.
- Unsupported leaf value flags are set.
- Keys are not strictly sorted by the shared comparator.
- Duplicate keys appear.

Empty leaf pages are valid. Empty values are valid. Empty keys are valid unless a
later implementation discovers a concrete frozen-format conflict; if rejected,
the rejection must be documented before tests are written.

### 5.3 Encode Rules

Leaf encoding must:

- Write all integer fields little-endian.
- Preserve the page id, page type, generation, and supported flags supplied by
  the caller.
- Emit slots in sorted key order.
- Pack key/value payloads without overlap.
- Reject an entry that cannot fit in an otherwise empty leaf page.
- Reject a page image whose slot directory and payload area would collide.
- Finalize the common-page checksum after the body is complete.

The encoder does not need to preserve byte-for-byte payload placement from a
decoded page. A decode/encode/decode round trip must preserve logical entries,
right-sibling reservation state, supported flags, and page header fields.

## 6. Branch Page Codec

### 6.1 Decoded Representation

Use a representation that makes the M0 branch shape explicit:

```zig
pub const BranchSlot = struct {
    left_child: PageId,
    separator: []const u8,
};

pub const DecodedBranch = struct {
    page_id: PageId,
    generation: u64,
    slots: []const BranchSlot,
    rightmost_child: PageId,
};
```

Each slot's `left_child` points to the subtree left of that separator.
`rightmost_child` is used when no separator is greater than the search key.

### 6.2 Decode Validation

Branch decoding must fail when:

- Common header checksum is invalid.
- Common header page type is not `PAGE_TYPE_BRANCH`.
- Encoded `page_id` does not match the physical page id.
- `key_count` cannot fit in the page after fixed fields and slot size.
- Slot directory extends outside the page.
- Any separator key range starts before the payload area, extends past the page,
  or overflows integer arithmetic.
- Separator payload ranges overlap forbidden regions or each other.
- Separator keys are not strictly sorted by the shared comparator.
- Any child page id is less than `FIRST_DATA_PAGE_ID`.
- Any child page id is equal to the branch page's own page id.
- Any child page id is duplicated within the branch page.
- `rightmost_child` is invalid by the same child-pointer rules.

The branch codec should validate local shape only. It should not read child pages
or prove child key ranges; that responsibility belongs to search and checker
layers.

### 6.3 Encode Rules

Branch encoding must:

- Write all integer fields little-endian.
- Emit separators in strict comparator order.
- Preserve the chosen branch split convention used by later mutation code.
- Pack separator payloads without overlap.
- Reject a branch image whose slot directory and payload area would collide.
- Reject invalid, duplicate, or self-referential child page ids.
- Finalize the common-page checksum after the body is complete.

M2 branch separators are routing lower bounds. Split code should later promote
the first key of the new right page. This feature must document that convention
in codec tests and helper names so later mutation code does not accidentally
switch to a different routing model.

## 7. Slot And Payload Validation

Implement shared validation helpers rather than duplicating arithmetic in leaf
and branch codecs.

Required helper behavior:

- Compute `slot_dir_start`, `slot_dir_len`, and `slot_dir_end` with checked
  arithmetic.
- Compute the minimum payload offset as the end of fixed body fields plus the
  slot directory.
- Treat every payload range as half-open: `[offset, offset + length)`.
- Reject ranges whose end overflows, whose start is before the payload area, or
  whose end is greater than `page_size`.
- Sort validated payload ranges by offset and reject overlap.
- Allow zero-length values.
- Allow zero-length keys only if the chosen key policy supports empty keys.
- Reject zero-length branch separators unless the comparator/key policy
  explicitly permits them and tests prove routing correctness.

The implementation should report precise internal codec errors, even if the
public API maps them to `Corruption`. Precise errors make malformed fixture tests
and checker diagnostics more useful.

## 8. Branch Routing

All branch traversal must use this rule:

1. Binary-search separators for the first separator key greater than the search
   key.
2. If such a separator exists, descend into that slot's `left_child`.
3. Otherwise, descend into `rightmost_child`.

Equivalent pseudocode:

```zig
pub fn childForKey(branch: DecodedBranch, key: []const u8) PageId {
    var lo: usize = 0;
    var hi: usize = branch.slots.len;
    while (lo < hi) {
        const mid = lo + (hi - lo) / 2;
        if (compare(key, branch.slots[mid].separator) == .lt) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    if (lo < branch.slots.len) return branch.slots[lo].left_child;
    return branch.rightmost_child;
}
```

Under this model, a separator is a lower bound for the subtree to its right. A
search key equal to a separator routes right, not left, because the first
separator greater than the search key is selected.

Routing tests must cover:

- key less than the first separator routes to slot `0`'s child
- key equal to each separator routes to the child on its right
- key between separators routes to the expected child
- key greater than all separators routes to `rightmost_child`
- binary separator keys with prefix relationships and high-bit bytes

## 9. Search Algorithm

Implement read-only point search as the first consumer of the codecs.

Suggested result types:

```zig
pub const SearchFrame = struct {
    page_id: PageId,
    child_index: usize,
};

pub const SearchResult = union(enum) {
    found: struct {
        leaf_page_id: PageId,
        entry_index: usize,
        value: []const u8,
        path: []const SearchFrame,
    },
    missing: struct {
        leaf_page_id: PageId,
        insert_index: usize,
        path: []const SearchFrame,
    },
};
```

Search flow:

1. Read the selected metadata root page id.
2. Reject a root page id outside metadata `page_count`.
3. Read the root page by whole-page positional I/O.
4. Validate checksum, page id, and page type before decoding the body.
5. If the page is a branch, decode it, choose the child with the branch routing
   rule, append a path frame, and continue.
6. If the page is a leaf, decode it and binary-search entries with the shared
   comparator.
7. Return `found` with the value slice or `missing` with the insertion position.

Search must not:

- Scan unreachable pages.
- Infer tree state from file length.
- Accept invalid checksums or mismatched page ids.
- Use linear search except in very small helper tests.
- Return borrowed value slices beyond the page-buffer lifetime unless the caller
  owns or pins that buffer.

The path returned by search should be designed for later mutation planning, but
this feature must keep the search operation read-only and side-effect-free.

## 10. Tests

### 10.1 Comparator Tests

- `orders_empty_and_prefix_keys`
- `orders_binary_keys_as_unsigned_bytes`
- `treats_equal_bytes_as_equal_across_buffers`
- `sorts_mixed_key_fixture`

### 10.2 Leaf Codec Tests

- `decodes_empty_leaf_page`
- `round_trips_single_leaf_entry`
- `round_trips_binary_keys_and_empty_values`
- `rejects_unsorted_leaf_keys`
- `rejects_duplicate_leaf_keys`
- `rejects_slot_directory_past_page_end`
- `rejects_payload_out_of_bounds`
- `rejects_overlapping_payloads`
- `rejects_unsupported_value_flags`
- `rejects_wrong_page_id_or_checksum`

### 10.3 Branch Codec Tests

- `round_trips_branch_with_one_separator`
- `round_trips_branch_with_binary_separators`
- `rejects_unsorted_branch_separators`
- `rejects_duplicate_branch_separators`
- `rejects_child_before_first_data_page`
- `rejects_self_child_pointer`
- `rejects_duplicate_child_pointer`
- `rejects_payload_out_of_bounds`
- `rejects_wrong_page_type_or_checksum`

### 10.4 Routing And Search Tests

- `routes_less_equal_between_and_greater_than_separators`
- `routes_prefix_and_high_bit_separator_keys`
- `searches_empty_root_leaf_as_missing`
- `finds_key_in_root_leaf`
- `returns_missing_insert_position_in_root_leaf`
- `descends_one_level_branch_to_leaf`
- `descends_multiple_branch_levels`
- `rejects_search_through_invalid_child_page_type`
- `rejects_search_through_bad_child_checksum`
- `does_not_read_unreachable_pages`

### 10.5 Fixture Guidance

Use small synthetic page sizes only if the M1 test harness supports them within
the frozen M0 page-size range. Otherwise, build full `4096` byte page fixtures
with helper builders so malformed offsets and checksums remain explicit.

Every malformed fixture should state which invariant it violates. Avoid tests
that depend on incidental encoder layout unless the behavior is part of the
codec contract.

## 11. Checker Handoff

This feature should hand the checker reusable primitives instead of forcing it
to decode pages differently.

Provide:

- Shared comparator module.
- Leaf and branch decoders with strict local validation.
- Accessors for decoded leaf keys, values, and flags.
- Accessors for decoded branch separators and child pointers.
- Branch routing helper used by both search and checker.
- Optional diagnostic details for codec failures.

The checker remains responsible for:

- Reachability from selected metadata.
- Child page type validation.
- Child key-range validation across pages.
- Detecting duplicate reachable tree pages.
- Verifying all reachable leaves are at the accepted depth.
- Root sanity rules.
- Accepting M2 underfull leaves only when routing remains valid.
- Allowing stale separators after delete only when every remaining key is still
  reachable through normal branch search.

The checker must not maintain its own comparator or alternate branch routing
rule. Divergence between search and checker would hide exactly the stale-routing
bugs M2 needs to catch.

## 12. Acceptance Criteria

This feature is complete when:

- One comparator defines all raw-byte key ordering.
- Leaf pages decode and encode according to the M0 slot-directory format.
- Branch pages decode and encode according to the M0 separator/child format.
- Slot directory and payload validation rejects out-of-bounds, overlapping, and
  unsupported encodings deterministically.
- Leaf decoding rejects unsorted or duplicate keys.
- Branch decoding rejects unsorted separators and invalid child pointers.
- Branch routing follows the first-separator-greater-than-search-key rule.
- Point search works for an empty root leaf, a root leaf with entries, a one-level
  branch tree, and a multi-level branch tree.
- Search validates page id, page type, and checksum before interpreting every
  page body.
- Search does not inspect unreachable pages or infer state from file length.
- Tests cover comparator, leaf codec, branch codec, routing, search success, and
  malformed page failures.
- The checker can reuse the same decoders, comparator, and routing helper without
  duplicating tree semantics.
- The implementation leaves clear seams for later M2 mutation planning, M3
  transactions, M4 fault injection, M5 cursors, M6 overflow/freelist reuse, and
  M7 verification.
