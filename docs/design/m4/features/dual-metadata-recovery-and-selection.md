# Dual Metadata Recovery And Selection

## Goal

Implement the M4 startup path that reads metadata slots A and B, validates them deterministically, and publishes exactly one committed metadata snapshot or returns the correct open-time error.

This feature must implement the M0 recovery rules from [transaction-recovery.md](../../m0/transaction-recovery.md) and stay within the M4 scope from [m4-implementation-plan.md](../m4-implementation-plan.md). It must not introduce new on-disk format fields, recovery heuristics, or slot-selection tie-breakers.

## Scope

This plan covers:

- header bootstrap before any page-sized read
- metadata slot decoding and validation
- generation ordering and equal-generation rejection
- `txn_id` monotonicity checks across valid slots
- root and freelist candidate invalidation as part of slot validity
- error mapping from internal validation failures to public `open()` results
- tests and acceptance criteria for the recovery selector

This plan does not cover:

- write-path metadata construction beyond the invariants recovery depends on
- full tree traversal or non-critical unreachable-page inspection
- user-facing repair or salvage flows

## Recovery Contract

Startup recovery has one job: identify the latest valid committed metadata state without guessing.

The selector must follow these fixed rules:

1. Validate the file header bootstrap first.
2. Read slot A from page `1` and slot B from page `2` using the decoded page size.
3. Validate each slot independently.
4. Discard invalid slots before any generation comparison.
5. If zero valid slots remain, return `Corruption`.
6. If one valid slot remains, select it.
7. If two valid slots remain, compare `generation`.
8. If generations are equal, return `Corruption`.
9. Select the slot with the larger `generation`.
10. If the larger `generation` has a lower `txn_id` than the older valid slot, return `Corruption`.

The implementation must never choose a slot using `txn_id`, physical slot order, file length, page write timing, root shape, freelist contents, or any other fallback heuristic.

## Header Bootstrap

`open()` must not read metadata or normal pages until the fixed `512`-byte header bootstrap prefix has been validated.

### Required Checks

The header bootstrap validator must verify:

- header magic
- supported format version
- expected endianness marker
- page size encoding and allowed range
- required feature flags
- reserved bits and reserved fields that must be zero
- header CRC32C over the bootstrap prefix excluding the checksum field

### Output

The header bootstrap stage should return a structured result that contains:

- decoded `page_size`
- decoded format version
- decoded feature flags
- a classification for any failure

Recovery must stop immediately on header failure. It must not attempt metadata reads with an untrusted page size.

### Error Mapping

- supported file shape but invalid bootstrap fields or checksum -> `Corruption`
- recognized file that requires unsupported version or required feature flags -> `IncompatibleVersion`

## Slot Validation

Each metadata slot must be validated in isolation before recovery compares the two slots.

### Physical Read Rules

- slot A must be read from page `1`
- slot B must be read from page `2`
- each page must be read using the validated header `page_size`
- short reads, torn reads, and decode failures must invalidate only the affected slot unless they prevent reading both slots entirely

### Required Metadata Checks

A slot is valid only if all of the following pass:

- metadata magic matches the frozen M0 value
- page type is metadata
- on-disk `slot_id` matches the physical slot being read
- metadata page size matches the header page size
- metadata format version is supported
- required feature flags are supported
- reserved fields that must be zero are zero
- metadata checksum matches
- `page_count >= 4`
- `root_page_id >= 3` and `root_page_id < page_count`
- `freelist_page_id >= 3` and `freelist_page_id < page_count`

Validation must produce per-slot diagnostics even when the other slot is good enough for recovery to continue.

## Root And Freelist Candidate Invalidation

Root and freelist validation is part of metadata validity, not a later refinement step.

After the metadata page itself passes structural checks, recovery must validate the critical pages referenced by that slot:

- read `root_page_id`
- read `freelist_page_id`
- verify each referenced page is readable
- verify each referenced page checksum
- verify root page type is `LEAF` or `BRANCH`
- verify freelist page type is `FREELIST`

If any of these checks fail, the metadata slot is invalid and must be discarded before generation comparison.

### Required Behavior

- a slot with a bad root must not compete in generation ordering
- a slot with a bad freelist must not compete in generation ordering
- if the newer-looking slot is invalid because its root or freelist is bad, recovery must fall back to the older valid slot
- recovery must not return `Corruption` solely because a higher raw `generation` exists in an already-invalid slot

## Generation Ordering

`generation` is the only slot ordering field.

### Ordering Rules

- if exactly one slot is valid, select it regardless of the other slot's raw generation value
- if both slots are valid and generations differ, select the larger generation
- if both slots are valid and generations are equal, return `Corruption`
- equal generations are always corruption, even if the slot bytes are identical

### Implementation Notes

The selector should compare generation only after both slots have been reduced to:

- `valid`
- `invalid_corruption`
- `invalid_incompatible`

This keeps selection logic small and prevents partial validation state from leaking into the chooser.

## Txn Monotonicity

`txn_id` is not a selector, but it must still be monotonic with committed generation order.

### Required Rule

If both slots are valid and one slot has a larger `generation`, that newer slot must not have a smaller `txn_id` than the older valid slot.

### Outcomes

- newer generation with equal `txn_id` -> allowed by recovery, though unusual
- newer generation with larger `txn_id` -> allowed
- newer generation with smaller `txn_id` -> `Corruption`

This matches the M4 plan language: `txn_id` is a monotonic consistency check, never a tie-breaker.

## Error Mapping

Recovery should keep internal validation reasons precise, but public `open()` results remain small and stable.

### Public Boundary Mapping

| Internal condition | Public result |
| --- | --- |
| header unsupported version or required feature flag | `IncompatibleVersion` |
| slot metadata unsupported version or required feature flag, and no compatible valid slot remains | `IncompatibleVersion` |
| header checksum failure | `Corruption` |
| header structural failure | `Corruption` |
| both slots invalid for corruption reasons | `Corruption` |
| one slot invalid, one slot valid | success with valid slot |
| both slots valid but equal generation | `Corruption` |
| both slots valid but newer slot has lower `txn_id` | `Corruption` |
| selected slot references invalid root or freelist | treat slot as invalid first; final result depends on whether another valid slot remains |

### Internal Classification Guidance

Implementation should preserve at least these internal classes:

- `invalid_header`
- `incompatible_header`
- `invalid_metadata_slot`
- `incompatible_metadata_slot`
- `metadata_equal_generation`
- `metadata_txn_regression`
- `critical_page_out_of_range`
- `critical_page_checksum_mismatch`
- `critical_page_type_mismatch`

The recovery path may surface them through an internal checker or debug logs, but `open()` should still collapse them into the public error set above.

## Suggested Implementation Shape

Use a small staged pipeline instead of mixing validation and selection in one function.

### Stage 1: Header Bootstrap

- read bootstrap prefix
- validate and decode header
- produce `page_size`

### Stage 2: Per-Slot Validation

For each physical slot:

- read metadata page
- decode and validate metadata fields
- validate referenced root
- validate referenced freelist
- return a structured slot result

### Stage 3: Selection

- collect only valid slots
- apply zero/one/two-slot decision tree
- run equal-generation and `txn_id` regression checks
- return selected metadata snapshot plus selected slot id

### Stage 4: Publication

- publish the selected metadata into the in-memory current database state
- do not publish any state before the selection result is final

## Tests

Tests should reopen through the normal startup path rather than calling the selector with synthetic in-memory pages only.

### Header Bootstrap Tests

| Case | Expected result |
| --- | --- |
| bad header magic | `Corruption` |
| bad header checksum | `Corruption` |
| unsupported major version | `IncompatibleVersion` |
| unsupported required feature flag | `IncompatibleVersion` |
| invalid page size encoding | `Corruption` |
| reserved bit set | `Corruption` |

### Slot Validation Tests

| Case | Expected result |
| --- | --- |
| slot A valid, slot B zeroed | select A |
| slot A checksum bad, slot B valid | select B |
| slot A wrong physical `slot_id`, slot B valid | select B |
| slot A short read, slot B valid | select B |
| both slots checksum bad | `Corruption` |
| both slots unsupported version | `IncompatibleVersion` |

### Generation And Txn Tests

| Case | Expected result |
| --- | --- |
| A valid with larger generation | select A |
| B valid with larger generation | select B |
| both valid, equal generation, different bytes | `Corruption` |
| both valid, equal generation, identical bytes | `Corruption` |
| newer generation with lower `txn_id` | `Corruption` |
| newer generation with equal `txn_id` | select newer generation |

### Root And Freelist Invalidation Tests

| Case | Expected result |
| --- | --- |
| newer slot root out of range, older slot valid | select older slot |
| newer slot freelist out of range, older slot valid | select older slot |
| newer slot root checksum bad, older slot valid | select older slot |
| newer slot freelist checksum bad, older slot valid | select older slot |
| newer slot root wrong type, older slot valid | select older slot |
| newer slot freelist wrong type, older slot valid | select older slot |
| both slots point to invalid critical pages | `Corruption` |

### Mixed Compatibility Tests

| Case | Expected result |
| --- | --- |
| one slot incompatible, one slot valid and supported | select supported valid slot |
| one slot incompatible, other slot corrupt | `IncompatibleVersion` only if no compatible valid slot remains and incompatibility is the only blocking reason |
| one slot incompatible older, one slot valid newer | select valid newer slot |

## Acceptance

- `open()` validates the fixed header bootstrap before any metadata page read.
- Header validation is the only source of trusted `page_size`.
- Metadata A and B are validated independently and produce per-slot outcomes.
- Slot validity includes root and freelist page validation.
- Invalid root or freelist candidates discard the referencing slot before generation comparison.
- Recovery selects by `generation` only after invalid slots have been discarded.
- Equal valid generations always return `Corruption`.
- `txn_id` never acts as a tie-breaker.
- A newer valid generation with a lower `txn_id` than the older valid slot returns `Corruption`.
- One valid slot is enough for successful open, regardless of the invalid slot's raw generation.
- Unsupported required version or feature combinations map to `IncompatibleVersion`.
- Structural corruption and checksum failures map to `Corruption`.
- The selected slot id is published alongside the selected metadata snapshot so later commits can alternate slots strictly.
- Tests cover header bootstrap, slot validation, generation ordering, `txn_id` monotonicity, root/freelist invalidation, and public error mapping.
