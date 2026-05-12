# Catalog Schema And Compatibility

## 1. Goal

This document turns the M5 catalog compatibility requirements into an implementation plan for the catalog root schema. It defines:

- the marker record stored in the catalog tree
- the fixed descriptor encoding for bucket catalog entries
- open-time handling for legacy unmarked roots
- how catalog validation maps to public error classes
- the tests required before the feature is considered complete
- the acceptance conditions for M5 catalog compatibility

This plan must preserve all M0 storage-format rules from [storage-format.md](../../m0/storage-format.md): no new normal page type, no metadata switch-point changes, no in-place mutation of visible pages, and no reinterpretation of unsupported files as valid catalog roots.

## 2. Scope And Invariants

The catalog schema is a logical layer on top of the existing M0 B+Tree page format.

Required invariants:

- `metadata.root_page_id` points to the catalog root B+Tree in M5 databases.
- The catalog tree uses ordinary `BRANCH` and `LEAF` pages only.
- Internal catalog records use the reserved `0x00` key prefix.
- User bucket names must never begin with `0x00`.
- `listBuckets()` must filter out all internal catalog records.
- A non-empty unmarked root must never be silently treated as a valid bucket catalog.
- Corruption in a supported catalog schema must map to `Corruption`.
- A newer required catalog schema or unsupported required feature must map to `IncompatibleVersion`.

## 3. Marker Record

### 3.1 Purpose

The marker record prevents accidental reinterpretation of legacy single-key-space trees as catalog trees. It also provides an in-tree compatibility check independent of header or metadata state.

### 3.2 Marker Key

The implementation should define one exact internal key constant and keep it next to the catalog encoder/decoder.

Recommended key bytes:

```text
00 63 61 74 61 6c 6f 67 2f 73 63 68 65 6d 61
```

This is the byte string `"\x00catalog/schema"`.

Rules:

- The key is reserved for internal use only.
- No user bucket name may equal or start with this internal prefix because all user names starting with `0x00` are already invalid.
- The key must sort before all user bucket names, but listing logic must still explicitly filter internal records instead of relying on sort position.

### 3.3 Marker Value Layout

The marker value must use a fixed-width little-endian binary encoding:

| Field | Type | Value in M5 |
| --- | --- | --- |
| `catalog_schema_version` | `u16` | `1` |
| `descriptor_encoding_version` | `u16` | `1` |
| `required_flags` | `u32` | `0` |
| `reserved0` | `u64` | `0` |
| `reserved1` | `u64` | `0` |

Encoded size: `24` bytes.

Validation rules:

- The value length must be exactly `24` bytes.
- `catalog_schema_version == 0` is invalid and maps to `Corruption`.
- `catalog_schema_version > supported_version` maps to `IncompatibleVersion`.
- `descriptor_encoding_version == 0` is invalid and maps to `Corruption`.
- `descriptor_encoding_version > supported_version` maps to `IncompatibleVersion`.
- Unknown required bits in `required_flags` map to `IncompatibleVersion`.
- Any non-zero reserved field maps to `Corruption`.

### 3.4 Write Rules

For new M5-created databases:

- The first committed catalog root must contain the marker record.
- The marker must be inserted before the metadata switch that makes the catalog visible.

For unmarked empty roots opened by an M5 engine:

- Read-only open may treat the root as an empty catalog in memory.
- The first write transaction that creates, drops, or otherwise mutates buckets must stage the marker record into the catalog tree before commit.
- That write transaction must publish both the marker and any bucket mutations atomically through the normal metadata switch.

## 4. Descriptor Encoding

### 4.1 Catalog Entry Model

Each user bucket name maps to a fixed-width descriptor value stored in the catalog B+Tree. The value must not depend on ad hoc struct layout or compiler packing.

### 4.2 Descriptor Layout

Each descriptor value must use this exact little-endian layout:

| Field | Type | Value / rule |
| --- | --- | --- |
| `descriptor_version` | `u16` | `1` |
| `descriptor_kind` | `u16` | `1` for user bucket |
| `flags` | `u32` | `0` in M5 unless explicitly defined later |
| `bucket_root_page_id` | `u64` | valid root page id |
| `entry_count` | `u64` | exact count or `UINT64_MAX` sentinel |
| `reserved0` | `u64` | `0` |
| `reserved1` | `u64` | `0` |

Encoded size: `40` bytes.

### 4.3 Descriptor Validation

Decoding must validate all of the following before a bucket handle is returned:

- value length is exactly `40` bytes
- `descriptor_version == 1`
- `descriptor_kind == 1`
- `flags` contains no unknown required bits
- `bucket_root_page_id >= 3`
- `bucket_root_page_id < page_count` from selected metadata
- `entry_count` is either an exact maintained count or `UINT64_MAX`
- `reserved0 == 0`
- `reserved1 == 0`

Failure mapping:

- malformed length or non-zero reserved fields: `Corruption`
- unsupported descriptor version or unknown required bits: `IncompatibleVersion`
- invalid root page id range in an otherwise supported file: `Corruption`

### 4.4 Empty Bucket Encoding

An empty bucket must still own a real leaf root page. The descriptor must never use `0` as a sentinel root id because M0 requires `root_page_id >= 3` for visible tree roots and page references must stay in-range.

Implementation rule:

- `createBucket()` allocates a new empty leaf page immediately.
- The descriptor stores that page id even when `entry_count == 0` or `UINT64_MAX`.

### 4.5 Determinism

Both marker and descriptor encoders must be byte-for-byte deterministic. The implementation should expose one encoder and one decoder per structure and use them in open, commit, tests, and corruption fixtures.

## 5. Legacy Unmarked Roots

### 5.1 Classification During Open

When opening the root tree selected by metadata, the engine must classify the catalog state in this order:

1. Root tree contains the marker record with a supported schema.
2. Root tree is empty and contains no marker.
3. Root tree is non-empty and contains no marker.
4. Root tree contains a malformed or unsupported marker.

### 5.2 Supported Cases

Case 1: marked root

- Treat as a normal M5 catalog root.
- Validate the marker before decoding any bucket descriptors.

Case 2: unmarked empty root

- Treat as a legacy empty root that can be upgraded lazily.
- Reads see an empty bucket catalog.
- The first bucket-management write must insert the marker before commit.

### 5.3 Rejected Case

Case 3: unmarked non-empty root

- Do not inspect normal records as if they were descriptors.
- Return a compatibility error immediately.
- Recommended public error class: `IncompatibleVersion`.
- If the implementation later adds a more specific internal cause such as `SchemaMigrationRequired`, it should still surface through the compatibility path rather than `Corruption`.

This rule is the main safety barrier against corrupting or misclassifying valid pre-M5 databases.

### 5.4 Malformed Or Unsupported Marker

Case 4 splits into two subcases:

- Supported schema bytes but malformed value, illegal reserved bits, duplicate conflicting marker, or impossible version fields: `Corruption`
- Marker or marker flags requiring a newer catalog schema than the engine supports: `IncompatibleVersion`

### 5.5 Open-Time Algorithm

Recommended implementation order:

1. Open database using existing M0 header and metadata validation.
2. Open the root B+Tree selected by `metadata.root_page_id`.
3. Probe the exact marker key.
4. If marker exists, decode and validate it.
5. If marker does not exist, determine whether the root tree is empty.
6. Empty and unmarked: set in-memory catalog state to `legacy_empty_unmarked`.
7. Non-empty and unmarked: fail open with `IncompatibleVersion`.
8. Marked and valid: set in-memory catalog state to `marked_v1`.

The empty/non-empty check should use the existing B+Tree cursor primitive and should not require a separate page-type or metadata flag.

## 6. Error Mapping

### 6.1 Public Error Classes

Catalog schema validation should map into the existing M0 compatibility policy:

- `Corruption`
- `IncompatibleVersion`

The feature may use more specific internal diagnostics, but these must collapse into the public classes above unless and until the public error model expands.

### 6.2 Mapping Table

| Condition | Public error |
| --- | --- |
| marker key missing in non-empty root | `IncompatibleVersion` |
| unmarked empty root | no error; treat as legacy-empty compatibility mode |
| marker length wrong | `Corruption` |
| marker reserved field non-zero | `Corruption` |
| marker schema version newer than supported | `IncompatibleVersion` |
| marker descriptor encoding version newer than supported | `IncompatibleVersion` |
| marker required flags contain unsupported required bit | `IncompatibleVersion` |
| descriptor length wrong | `Corruption` |
| descriptor reserved field non-zero | `Corruption` |
| descriptor version newer than supported | `IncompatibleVersion` |
| descriptor kind unknown but marked required for current schema | `IncompatibleVersion` |
| descriptor root page id out of metadata range | `Corruption` |
| duplicate user bucket key in logically inconsistent catalog scan | `Corruption` |
| internal record exposed through user listing | implementation bug; test failure |

### 6.3 Error Handling Principles

- Prefer `IncompatibleVersion` when the bytes could be valid for a newer engine.
- Prefer `Corruption` when the bytes violate the supported schema's explicit invariants.
- Do not map non-empty unmarked roots to `Corruption`; the file may be healthy, just not safely interpretable as an M5 catalog.

## 7. Implementation Steps

1. Add catalog constants for the marker key, marker value size, descriptor value size, and supported schema versions.
2. Implement deterministic marker encode/decode helpers with strict validation.
3. Implement deterministic descriptor encode/decode helpers with strict validation.
4. Add catalog-open classification for `marked_v1`, `legacy_empty_unmarked`, and reject `non_empty_unmarked`.
5. Thread catalog compatibility state through database open so write transactions know whether they must inject the marker before first bucket mutation commit.
6. Ensure `createBucket()`, `createBucketIfMissing()` when creating, and `dropBucket()` operate only after the write transaction has staged a valid marker record.
7. Ensure `listBuckets()` and any future catalog iteration path filter internal `0x00` records explicitly.
8. Add corruption and compatibility fixtures for malformed marker and descriptor cases.
9. Re-run M0-M4 open, recovery, and transaction tests to confirm no regression in metadata switching or crash behavior.

## 8. Tests

### 8.1 Marker Record Tests

- Create a new M5 database and assert the committed catalog root contains the exact marker key and a valid `24`-byte marker value.
- Assert `listBuckets()` returns no internal marker entry.
- Corrupt marker length and assert open returns `Corruption`.
- Corrupt marker reserved fields and assert open returns `Corruption`.
- Raise marker schema version above supported and assert open returns `IncompatibleVersion`.
- Raise marker descriptor encoding version above supported and assert open returns `IncompatibleVersion`.
- Set an unsupported required flag and assert open returns `IncompatibleVersion`.

### 8.2 Descriptor Tests

- Encode and decode a valid descriptor round-trip with a normal bucket root page id.
- Reject descriptor values shorter or longer than `40` bytes with `Corruption`.
- Reject non-zero reserved fields with `Corruption`.
- Reject unsupported descriptor version with `IncompatibleVersion`.
- Reject unsupported descriptor kind with `IncompatibleVersion`.
- Reject root page ids `< 3` or `>= page_count` with `Corruption`.
- Verify empty buckets still reference a valid allocated leaf root page.

### 8.3 Legacy Compatibility Tests

- Open an unmarked empty root and verify reads behave as an empty catalog.
- Start a write transaction on an unmarked empty root, create a bucket, commit, reopen, and verify the marker is now present.
- Open an unmarked non-empty root and assert `IncompatibleVersion`.
- Verify the non-empty legacy case does not reinterpret the first user key/value pair as a bucket descriptor.

### 8.4 Error-Mapping Tests

- Build focused fixtures for each row in the error-mapping table and assert the public error class exactly matches the spec.
- Ensure malformed supported-schema marker bytes do not leak as generic I/O or state errors.
- Ensure newer-schema marker bytes do not leak as `Corruption`.

### 8.5 Integration Tests

- Create several buckets, commit, reopen, and verify descriptors still decode correctly.
- Drop and recreate buckets across commits and reopens to confirm catalog entries continue to use deterministic encoding.
- Inject failures before metadata publish and confirm recovery chooses either the old root or the fully marked new root, never a half-upgraded catalog.
- Run concurrent-reader tests where an older reader sees the pre-upgrade empty root while a writer commits the first marker plus bucket creation.

## 9. Acceptance

This feature is complete when all of the following are true:

- New M5 databases always commit a catalog root with the exact marker record.
- Marker and descriptor values use fixed-width little-endian deterministic encodings.
- User bucket names beginning with `0x00` are rejected.
- Internal catalog records are hidden from `listBuckets()` and any user-facing catalog enumeration.
- Opening an unmarked empty root is safe and upgrades on the first bucket write.
- Opening an unmarked non-empty root fails with `IncompatibleVersion`.
- Malformed marker or descriptor data in a supported schema fails with `Corruption`.
- Newer required marker or descriptor versions fail with `IncompatibleVersion`.
- Descriptor root page ids are validated against selected metadata `page_count`.
- Empty buckets always store a valid leaf root page id rather than a sentinel.
- Crash recovery preserves the M0 rule that metadata is the only committed-state switch point, including the first marker insertion.
- Existing M0-M4 open, recovery, and transaction guarantees remain intact after the catalog layer is introduced.
