# M6 Feature Plan: Overflow Reference Encoding And Chain Layout

This document defines the concrete M6 implementation plan for encoding overflow
references inside leaf pages and laying out `PAGE_TYPE_OVERFLOW` chains on disk.
It binds the M0 storage-format reservations in
[`docs/design/m0/storage-format.md`](/D:/代码D/zi6db/docs/design/m0/storage-format.md)
to one stable implementation path that preserves leaf slot compatibility.

This feature covers only:

- leaf flag binding for overflow values
- the overflow reference record stored in leaf payload space
- the body layout for overflow pages
- validation rules for read, write, delete, and recovery paths
- tests and acceptance criteria for this feature

It does not redefine leaf slot structure, page header structure, commit
ordering, or freelist ownership rules from M0 and the main
[`m6-implementation-plan.md`](/D:/代码D/zi6db/docs/design/m6/m6-implementation-plan.md).

## 1. Scope And Goals

The implementation is complete when a leaf entry can represent either:

- an inline value stored directly in leaf payload bytes, or
- an overflow reference record that points to one logical value chain stored in
  one or more overflow pages

The design must preserve these M0 constraints:

- leaf slots keep the existing `key offset`, `key length`, `value offset`,
  `value length`, and `flags` fields
- `PAGE_TYPE_OVERFLOW` remains the only page type for spilled values
- overflow pages are copy-on-write and belong to exactly one logical value
  version
- metadata remains the only commit switch point

## 2. Leaf Flag Binding

### 2.1 Flag Contract

M6 binds one reserved leaf value flag as `VALUE_FLAG_OVERFLOW`.

When `VALUE_FLAG_OVERFLOW` is clear:

- `value_offset` points to inline user value bytes
- `value_length` is the inline value byte length
- the payload bytes at `value_offset..value_offset+value_length` are returned
  directly as the logical value

When `VALUE_FLAG_OVERFLOW` is set:

- `value_offset` points to an overflow reference record inside the leaf payload
  area
- `value_length` is the encoded byte length of that overflow reference record
- the bytes at that location are not user value bytes and must never be exposed
  directly through the public value API

### 2.2 Binding Rules

Implementation rules:

- The flag meaning is binary: a leaf value is either inline or overflow-backed.
- No mixed encoding is allowed for one leaf entry.
- `value_length` for an overflow-backed entry must equal the encoded reference
  record size exactly.
- The page builder must reject any overflow-backed leaf slot whose
  `value_length` does not match the reference record version it writes.
- Readers must branch on the flag before interpreting `value_offset` and
  `value_length`.

### 2.3 Placement Policy Boundary

The decision to spill a value to overflow is an implementation policy, not a
format invariant. The feature only requires that once written:

- an inline entry keeps inline semantics until replaced
- an overflow entry keeps overflow-reference semantics until replaced

## 3. Overflow Reference Record

### 3.1 Record Location

The overflow reference record lives inside the leaf payload area and is covered
by the leaf page checksum like any other leaf payload bytes.

The leaf slot references it through `value_offset` and `value_length`. No extra
slot fields are introduced.

### 3.2 Record Fields

The initial record layout should contain these fields in little-endian order:

1. `first_overflow_page_id: u64`
2. `overflow_page_count: u32`
3. `logical_value_length: u64`
4. `record_flags: u32`

`record_flags` is reserved and must be written as zero in M6.

This gives the implementation a fixed-width record with enough data to:

- find the first page of the chain
- know the exact number of pages expected
- know the exact logical value length to materialize
- keep reserved space for future extensions without changing leaf slot shape

### 3.3 Record Invariants

Required invariants:

- `first_overflow_page_id >= 3`
- `overflow_page_count >= 1`
- `logical_value_length >= 1` for overflow-backed values
- `record_flags == 0` in M6
- the record must fit completely inside the leaf payload area
- the record must not overlap key bytes or another value payload region

### 3.4 Encoding And Decoding Responsibilities

Writer responsibilities:

- encode the record only after the complete overflow chain has been allocated
  and assigned final page IDs
- write the record into the cloned leaf payload before the leaf checksum is
  computed
- ensure `overflow_page_count` matches the number of pages actually written

Reader responsibilities:

- validate the leaf page checksum first
- inspect `VALUE_FLAG_OVERFLOW`
- decode the reference record only if the flag is set
- treat malformed record fields as `Corruption`

Delete and replace responsibilities:

- decode and validate the record before attempting to enumerate pages to free
- fail with `Corruption` if the record cannot be trusted enough to walk the
  chain exactly

## 4. Overflow Page Body

### 4.1 Body Purpose

Each overflow page must be self-describing enough that the reader can validate
page ordering, chain ownership, and payload contribution without consulting
ambient state beyond the leaf reference and selected metadata snapshot.

### 4.2 Required Body Fields

After the common normal-page header, each overflow page body should store:

1. `chain_first_page_id: u64`
2. `index_in_chain: u32`
3. `next_overflow_page_id: u64`
4. `payload_length: u32`
5. `logical_value_length: u64` on page index `0`, otherwise `0`
6. reserved bytes set to zero
7. payload bytes

Field semantics:

- `chain_first_page_id` identifies the chain owner and must equal the first
  page ID for every page in the chain.
- `index_in_chain` starts at `0` on the first page and increments by `1`.
- `next_overflow_page_id == 0` marks the final page.
- `payload_length` is the number of valid payload bytes stored on this page.
- `logical_value_length` is repeated only on the first page to anchor chain
  validation. Non-first pages store `0` for this field in M6.

### 4.3 Layout Rules

Implementation rules:

- usable payload capacity is `page_size - common_header_size - overflow_body_prefix_size`
- `payload_length` must be `> 0`
- `payload_length` must be `<= usable payload capacity`
- all reserved bytes must be zero before checksum calculation
- the final page may have a short payload
- all non-final pages should be filled to usable payload capacity unless the
  writer intentionally keeps a simpler builder contract

### 4.4 Chain Construction Rules

Writers should build chains in this order:

1. split the logical value into page-sized payload fragments
2. allocate page IDs for the full chain
3. assign `chain_first_page_id` to the first allocated page ID
4. encode each page with its index, next page ID, and payload fragment
5. write and checksum all overflow pages
6. encode the leaf reference record that points to the first page

Correctness rules:

- chain correctness must not depend on contiguous page IDs
- a chain belongs to exactly one committed logical value version
- partial chain allocation or encoding failure must not leak a published leaf
  reference

## 5. Validation Rules

### 5.1 Leaf-Level Validation

When a leaf slot has `VALUE_FLAG_OVERFLOW` set, validation must confirm:

- `value_offset` and `value_length` stay within the leaf payload bounds
- `value_length` equals the fixed overflow reference record size
- the decoded `first_overflow_page_id` is `< selected_metadata.page_count`
- `overflow_page_count >= 1`
- `logical_value_length >= 1`
- reserved `record_flags` bits are zero

### 5.2 Chain Validation During Read

Reading an overflow-backed value must validate:

- the first page exists at `first_overflow_page_id`
- every page has `page_type = OVERFLOW`
- every page checksum is valid before payload use
- every page header `page_id` matches the physical page being read
- every page has `chain_first_page_id == first_overflow_page_id`
- the first page has `index_in_chain == 0`
- each later page increments `index_in_chain` by exactly `1`
- non-final pages have `next_overflow_page_id != 0`
- the final page has `next_overflow_page_id == 0`
- no page ID repeats within the same chain walk
- each page ID is `< selected_metadata.page_count`
- each `payload_length` is within overflow-page usable capacity
- the first page stores `logical_value_length` equal to the leaf reference
- non-first pages store `logical_value_length == 0`
- the number of pages walked equals `overflow_page_count`
- the sum of `payload_length` values equals `logical_value_length`

Any violation returns `Corruption`. The reader must never return partial value
bytes from a malformed chain.

### 5.3 Validation During Delete And Replace

Delete and replace paths must validate enough of the chain to enumerate the
exact page set being retired.

Required behavior:

- walk the full chain from the leaf reference
- reject cycles, early termination, extra pages, out-of-range page IDs, and
  length mismatches
- add pages to `pending_free[current_txn_id]` only after full validation

If validation fails:

- the write operation fails with `Corruption`
- the leaf entry is not removed or rewritten
- the engine does not guess which pages were intended to belong to the chain

### 5.4 Recovery And Open Boundaries

`open()` is not required to traverse every live overflow chain in the database.
However, this feature requires:

- metadata validation to preserve the selected `page_count` boundary
- any later read of a reachable overflow value to perform full chain validation
- reusable validation helpers so M7 verify can traverse live chains without
  duplicating parser logic

## 6. Implementation Sequence

1. Define the fixed-width overflow reference record encoder and decoder.
2. Bind `VALUE_FLAG_OVERFLOW` in leaf value decode and encode paths.
3. Add overflow page body encoder and decoder with checksum-covered reserved
   bytes zeroing.
4. Implement chain writer construction from a fully materialized logical value.
5. Implement chain walker validation for read paths.
6. Reuse the same chain walker for delete and replace enumeration.
7. Expose internal helpers that M7 verification can call later.

## 7. Tests

### 7.1 Encoding Tests

- Inline leaf entries keep existing value semantics when
  `VALUE_FLAG_OVERFLOW` is clear.
- Overflow-backed leaf entries encode the fixed reference record size exactly.
- Overflow reference fields round-trip through page encode and decode.
- Reserved record flags are written as zero.

### 7.2 Chain Layout Tests

- A one-page overflow value produces `overflow_page_count = 1`,
  `index_in_chain = 0`, and `next_overflow_page_id = 0`.
- A multi-page overflow value produces a monotonic chain with correct next-page
  links and correct first-page ownership on every page.
- Non-contiguous page IDs still decode correctly.
- The final page stores only the tail payload fragment and terminates the chain.

### 7.3 Boundary Tests

- The largest value that stays inline still decodes inline.
- The smallest value that spills to overflow decodes through exactly one chain.
- A value that spans several overflow pages reopens and round-trips exactly.
- Replacing an overflow value with a smaller inline value frees the old chain
  through the normal pending-free path.
- Replacing an inline value with an overflow value creates a valid chain and
  does not corrupt leaf layout.

### 7.4 Corruption Tests

- Leaf slot sets `VALUE_FLAG_OVERFLOW` but `value_length` is not the reference
  size.
- Overflow reference points to a non-overflow page.
- Overflow page checksum is invalid.
- `chain_first_page_id` differs from the leaf reference.
- `index_in_chain` skips or repeats.
- The chain terminates early.
- The chain is longer than `overflow_page_count`.
- `payload_length` exceeds usable capacity.
- Summed payload bytes do not equal `logical_value_length`.
- The chain contains a cycle.

Each case must fail with `Corruption` and must not expose partial user data.

### 7.5 Delete And Replace Tests

- Deleting an overflow-backed key validates and frees the exact referenced page
  set.
- Replacing one overflow chain with another retires only the old chain pages.
- Delete of a corrupt overflow chain fails and keeps the logical key unchanged.
- Replace of a corrupt overflow chain fails and does not publish a new value.

## 8. Acceptance

This feature is accepted when all of the following are true:

- `VALUE_FLAG_OVERFLOW` is bound to one stable leaf value interpretation.
- Leaf slot structure remains unchanged from M0.
- Overflow reference records are stored in leaf payload space and protected by
  the leaf checksum.
- The reference record includes `first_overflow_page_id`,
  `overflow_page_count`, and `logical_value_length`.
- Overflow page bodies encode chain ownership, chain index, next-page linkage,
  and per-page payload length.
- Readers validate full chain structure before returning overflow-backed value
  bytes.
- Delete and replace paths validate the full old chain before freeing pages.
- Malformed records or chain pages fail with `Corruption`.
- Tests cover single-page, multi-page, non-contiguous, boundary, replace,
  delete, and corruption cases.
- The helpers introduced here are reusable by later M7 verification work
  without changing on-disk semantics.
