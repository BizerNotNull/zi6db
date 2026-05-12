# Metadata Root Freelist Initialization

This plan covers the M1 implementation slice that creates the smallest valid on-disk database state: metadata slot A, metadata slot B, the empty root leaf, and the empty freelist page. It depends on the frozen M0 format and recovery rules in `docs/design/m0/storage-format.md` and `docs/design/m0/transaction-recovery.md`.

## Scope

M1 must initialize and validate only the bootstrap state needed for `create()` and `open()`.

In scope:

- encode metadata slot A as the only valid initial metadata page
- encode metadata slot B as zeroed or otherwise invalid
- encode an empty root leaf page at page `3`
- encode an empty freelist page at page `4`
- compute and validate CRC32C checksums for metadata and normal pages
- set `page_count = 5` and reject selected metadata that claims pages beyond complete file length
- require each metadata page `slot_id` to match its physical page
- add tests for fresh creation, reopen, corruption, and future handoff invariants

Out of scope:

- B+Tree search, insert, split, delete, or root replacement
- non-empty freelist reuse or pending-free migration
- real transaction commit switching
- crash-fault injection beyond deterministic corruption/open tests
- overflow value handling

## Initial Page Layout

`create()` must write exactly the bootstrap pages required by M0:

| Page id | Role | Initial validity |
| --- | --- | --- |
| `0` | immutable header | valid |
| `1` | metadata slot A | valid |
| `2` | metadata slot B | invalid, preferably all zeroes |
| `3` | root leaf | valid empty leaf |
| `4` | freelist | valid empty freelist |

The initial metadata values are:

| Field | Value |
| --- | --- |
| `slot_id` | `A` |
| `generation` | `1` |
| `txn_id` | `1` |
| `root_page_id` | `3` |
| `freelist_page_id` | `4` |
| `page_count` | `5` |

The initial file length should be exactly `page_size * 5` after successful strict creation. Later open logic may tolerate complete or partial unreachable tail bytes beyond selected `page_count`, but this initializer must not create them deliberately.

## Metadata A Initialization

Implement a metadata encoder in the format layer that receives a strongly typed metadata struct and a physical slot id. For initial creation it must build slot A with:

- metadata magic set to the frozen M0 metadata magic
- page type set to `META`
- format major/minor copied from the validated runtime constants used by the header encoder
- page size copied from the create options after bounds validation
- `slot_id = A`
- `generation = 1`
- `txn_id = 1`
- `root_page_id = FIRST_DATA_PAGE_ID`
- `freelist_page_id = FIRST_DATA_PAGE_ID + 1`
- `page_count = FIRST_DATA_PAGE_ID + 2`
- state flags set only to supported M1 defaults
- all reserved fields zeroed
- metadata checksum written last

The checksum must cover the full metadata page except the checksum field itself. The encoder should zero the checksum field before computing CRC32C so tests can reproduce exact bytes.

## Metadata B Initialization

Slot B must not be valid after initial creation. Prefer writing a full zero page at physical page `2`.

This prevents startup from seeing two valid metadata pages with equal `generation = 1`, which M0 defines as corruption even if the contents are identical. It also exercises the normal initial recovery path: metadata A valid, metadata B invalid, select A.

Do not encode a valid slot B with a lower generation during M1 creation. The first valid slot B belongs to the first real commit in M3.

## Empty Root Leaf

The root page at page `3` must be encoded as a normal page using the common normal-page header:

- `page_id = 3`
- `page_type = LEAF`
- `page_generation = 1`
- flags set to supported M1 defaults
- checksum written last

The leaf body must represent an empty B+Tree leaf:

- `entry_count = 0`
- right-sibling reserved field set to the no-sibling sentinel or zero, whichever the M0 constants freeze
- slot directory empty
- key/value payload area empty
- all required-zero reserved bytes zeroed

M1 should expose only structural encode/validate helpers for this page. It must not add search or mutation logic to `db.open()`.

## Empty Freelist

The freelist page at page `4` must also use the common normal-page header:

- `page_id = 4`
- `page_type = FREELIST`
- `page_generation = 1`
- flags set to supported M1 defaults
- checksum written last

The body must preserve the persisted shape M6 will extend:

- `reusable_count = 0`
- no reusable page ids
- `pending_group_count = 0`
- no `pending_free` groups
- compaction/chunking reserved fields zeroed
- all future-use reserved bytes zeroed

Open validation must reject any selected freelist page with the wrong page type, a checksum mismatch, non-zero required-zero reserved fields, or body counters that exceed the page boundary.

M1 must not infer free pages from file tail residue. If a file has complete extra pages or partial tail bytes beyond selected `page_count`, open may ignore them, but they must not be inserted into this freelist.

## Checksum Rules

Use CRC32C consistently with the M0 checksum contracts:

- metadata checksum covers the whole metadata page except the metadata checksum field
- normal-page checksum covers `page_id`, `page_type`, `page_generation`, flags, and the page body while excluding the checksum field itself
- checksum fields are zeroed during calculation
- checksum mismatch on metadata, root, or freelist maps to public `Corruption`

Add test helpers that can corrupt one byte in each checksum-covered region and assert deterministic failure.

## Open-Path Validation

Metadata validation is candidate-local until final selection. For each metadata slot:

1. Decode from the physical page implied by the slot.
2. Reject if metadata magic or page type is invalid.
3. Reject if `slot_id` does not match the physical page: A on page `1`, B on page `2`.
4. Reject if page size, format version, or required feature flags disagree with the validated header.
5. Reject if required-zero reserved fields are non-zero.
6. Verify the metadata checksum.
7. Reject if `page_count < 5` for the M1 bootstrap shape.
8. Reject if `page_count` exceeds the complete whole-page file length.
9. Reject if `root_page_id` or `freelist_page_id` is below `FIRST_DATA_PAGE_ID` or greater than or equal to `page_count`.
10. Validate the referenced root page type and checksum.
11. Validate the referenced freelist page type, checksum, and empty-body structural fields.

After candidate validation:

- if only A is valid, select A
- if only B is valid, select B
- if neither is valid, return `Corruption`
- if both are valid with equal generation, return `Corruption`
- if both are valid and the newer generation has a lower `txn_id`, return `Corruption`
- otherwise select the higher `generation`

Record the selected slot id in `DB` state so M3 can write the inactive slot without rediscovering it.

## Page Count And File Length

`page_count` is the exclusive upper bound of allocated page ids. The initial value is `5`, covering pages `0` through `4`.

Open must compute:

- `complete_page_count = file_length / page_size`
- `partial_tail_bytes = file_length % page_size`

Validation rules:

- selected metadata is corrupt if `page_count > complete_page_count`
- `partial_tail_bytes` do not create an addressable page
- complete pages beyond selected `page_count` are unreachable tail residue
- unreachable tail residue is ignored and never added to the freelist during M1

## Implementation Steps

1. Add constants for the initial root page id, initial freelist page id, and initial page count near the existing format constants.
2. Implement metadata encode/decode helpers with physical-slot validation.
3. Implement empty leaf encode/validate helpers for the root page.
4. Implement empty freelist encode/validate helpers that preserve reusable and `pending_free` sections.
5. Wire `create()` to write header, metadata A, zero metadata B, root leaf, and freelist in page-id order.
6. Compute checksums after all page bytes and reserved fields are initialized.
7. Wire `open()` candidate validation to validate root and freelist pages before selecting metadata.
8. Store selected metadata and selected slot id in `DB` state.
9. Keep all validation errors detailed internally but map public lifecycle failures to M0 categories.

## Tests

Add automated tests with behavior-focused names:

- creating a database writes five complete pages
- reopening a fresh database selects metadata slot A
- fresh metadata has `generation = 1`, `txn_id = 1`, `root_page_id = 3`, `freelist_page_id = 4`, and `page_count = 5`
- metadata slot B is invalid on first creation
- metadata slot B with the wrong physical `slot_id` is rejected
- metadata checksum corruption invalidates that candidate
- both metadata pages invalid returns `Corruption`
- equal-generation valid metadata A and B returns `Corruption`
- newer metadata with lower `txn_id` returns `Corruption`
- root page checksum corruption returns `Corruption`
- root page type mismatch returns `Corruption`
- freelist page checksum corruption returns `Corruption`
- freelist page type mismatch returns `Corruption`
- selected `page_count` beyond complete file pages returns `Corruption`
- complete extra tail pages beyond selected `page_count` are ignored
- partial tail bytes beyond selected `page_count` are ignored
- empty freelist validates with zero reusable pages and zero pending-free groups

Where possible, use table-driven fixtures that mutate one field at a time. This lets M4 reuse the same corruption cases for crash-recovery fixtures.

## Handoff Notes

M2 handoff:

- the empty root page must already use the final `LEAF` page type and checksum shape
- leaf validation and page offset helpers should live in the format/storage layers, not in B+Tree mutation code
- M2 may add search and mutation on top of this empty leaf without changing the initial disk bytes

M6 handoff:

- the freelist body must preserve both persisted `reusable` and `pending_free` sections even when empty
- M6 owns non-empty freelist reuse, overflow pages, and restart-time pending-free migration
- M1 must not reclaim unreachable tail pages, because M6 recovery must trust the selected freelist instead of guessing

M4 handoff:

- metadata candidate validation must be deterministic and independently testable
- corruption tests should distinguish candidate invalidation from final open failure
- open must not special-case freshly created files; it should use the same validation path M4 crash fixtures will exercise
- strict create must flush initialized pages according to the M1 durability contract before reporting success

## Acceptance Checklist

- metadata A is the only valid metadata page after creation
- metadata B is zeroed or otherwise invalid
- root page `3` is a valid empty leaf with a valid checksum
- freelist page `4` is a valid empty freelist with a valid checksum
- metadata A points to root page `3`, freelist page `4`, and `page_count = 5`
- metadata `slot_id` must match the physical page during validation
- metadata/header duplicated fields must agree
- selected root and freelist pages are validated before open succeeds
- selected `page_count` must be backed by complete pages in the file
- unreachable tail bytes or pages are ignored and not treated as freelist entries
- selected metadata slot id is available in memory for the first M3 commit
