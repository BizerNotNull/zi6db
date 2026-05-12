# Header Codec And Validation Implementation Plan

This feature implements the M1 header bootstrap codec for `zi6db`. It turns the frozen M0 header rules into concrete encode, decode, checksum, validation, and error-mapping behavior used by `create()` and `open()`.

The header is immutable after database creation. Metadata must never be used to repair or override a failed header.

## Goals

- Encode a stable page `0` bootstrap header when creating a new database file.
- Decode and validate exactly the first `512` bytes before any page-sized structure is interpreted.
- Return the validated `page_size` only after all bootstrap checks pass.
- Reject non-database files, corrupted headers, unsupported versions, and unsupported required features with deterministic public errors.
- Keep all header byte layout and validation logic isolated from file I/O and database lifecycle policy.
- Provide field-level tests that later M1 metadata/open tests can reuse as fixtures.

## Module Boundary

The recommended implementation target is `src/format/header.zig`, backed by constants from `src/format/constants.zig`.

| Module | Owns | Must Not Own |
| --- | --- | --- |
| `format/constants.zig` | frozen M0 constants, selected byte values for magic, version, page-size limits, checksum algorithm identifiers | file handles, allocation policy, lifecycle decisions |
| `format/header.zig` | header struct, byte offsets, encode/decode helpers, checksum calculation, field validation, private validation errors | positional reads/writes, metadata selection, root/freelist validation |
| `storage/file.zig` | reading exactly `HEADER_BOOTSTRAP_PREFIX_SIZE` bytes from offset `0` and surfacing I/O failures | interpreting header bytes or choosing public format errors |
| `db.zig` | create/open orchestration, public error conversion, using decoded page size for later metadata reads | duplicating header byte layout or checksum logic |

The header codec should expose a small API, for example:

```zig
pub const Header = struct {
    format_major: u16,
    format_minor: u16,
    page_size: u32,
    required_features: u64,
    optional_features: u64,
};

pub fn encode(header: Header, out: *[HEADER_BOOTSTRAP_PREFIX_SIZE]u8) HeaderEncodeError!void;
pub fn decodeAndValidate(bytes: *const [HEADER_BOOTSTRAP_PREFIX_SIZE]u8) HeaderValidationError!Header;
pub fn checksum(bytes: *const [HEADER_BOOTSTRAP_PREFIX_SIZE]u8) u32;
```

Exact Zig signatures may vary, but only the format layer should know field offsets and checksum coverage.

## Encoding And Decoding Fields

M1 must bind exact byte sequences and offsets in code. After the first file is emitted, those values become stable format data and must not change without a format-version upgrade.

| Field | Direction | Validation |
| --- | --- | --- |
| `HEADER_MAGIC` | encode/decode | must identify a `zi6db` header; mismatch on a recognizable bootstrap is `Corruption` |
| `format_major` | encode/decode | must equal supported `FORMAT_MAJOR = 0`; mismatch is `IncompatibleVersion` |
| `format_minor` | encode/decode | must be `<= FORMAT_MINOR = 1`; future minor is `IncompatibleVersion` |
| `endianness_marker` | encode/decode | must indicate little-endian; any other value is `Corruption` |
| `page_size` | encode/decode | must be between `4096` and `65536`, inclusive, and aligned to itself for page offset math |
| `required_features` | encode/decode | unsupported required bits are `IncompatibleVersion` |
| `optional_features` | encode/decode | unknown optional bits may be preserved or ignored, but must not be treated as required |
| reserved bytes/bits | encode/decode | must encode as zero and validate as zero |
| `header_checksum` | encode/decode | CRC32C over the bootstrap prefix excluding the checksum field itself |

The header must not encode mutable runtime fields:

- `generation`
- `txn_id`
- `root_page_id`
- `freelist_page_id`
- `page_count`
- selected metadata slot

If later metadata duplicates any immutable header field, the metadata value must match the validated header exactly. Disagreement is metadata/header corruption, not a tie-breaker.

## Validation Order

`open()` should call the header codec in this order:

1. Read exactly `HEADER_BOOTSTRAP_PREFIX_SIZE` bytes from offset `0`.
2. If fewer than `512` bytes are available, classify the input as not enough structure for a bootstrap and return `FormatError`.
3. Check the bootstrap shape and magic bytes before reading version-dependent fields.
4. If the file is clearly not a `zi6db` bootstrap, return `FormatError`.
5. If the magic is close enough to identify a `zi6db` bootstrap but is malformed, return `Corruption`.
6. Decode fixed fields using little-endian integer reads only.
7. Validate `format_major`, `format_minor`, and unsupported required feature bits.
8. Validate the endianness marker.
9. Validate `page_size` against `MIN_PAGE_SIZE = 4096` and `MAX_PAGE_SIZE = 65536`.
10. Validate required-zero reserved fields and reserved feature bits.
11. Recompute the header checksum and compare it with the stored checksum.
12. Return a decoded `Header` only after every check has passed.

The codec must not read metadata, infer page size from file length, or attempt repair during this sequence.

## Checksum Rules

- Algorithm: `CRC32C`.
- Coverage: exactly the `512`-byte bootstrap prefix, excluding the bytes occupied by `header_checksum`.
- Encoding: the stored checksum is little-endian.
- Creation: zero or skip the checksum field while computing, write the final CRC32C once all other fields are encoded.
- Validation: compute with the checksum field excluded in the same way as creation, then compare against the stored value.
- Failure: header checksum mismatch maps to public `Corruption`.

The header checksum is independent from metadata and normal-page checksums. A valid metadata checksum cannot compensate for an invalid header checksum.

## Error Mapping

The format layer may return detailed internal errors, but public lifecycle calls should map them consistently.

| Condition | Public Error |
| --- | --- |
| file is shorter than the bootstrap prefix | `FormatError` |
| bytes do not identify a `zi6db` bootstrap | `FormatError` |
| recognizable `zi6db` bootstrap with malformed magic or shape | `Corruption` |
| unsupported `format_major` | `IncompatibleVersion` |
| future unsupported `format_minor` | `IncompatibleVersion` |
| unsupported required feature flag | `IncompatibleVersion` |
| unsupported endianness marker | `Corruption` |
| invalid page size | `Corruption` |
| non-zero required-zero reserved field | `Corruption` |
| header checksum mismatch | `Corruption` |
| required header read fails for OS reasons | `IoError` |

Suggested private validation errors include `NotDatabase`, `MalformedMagic`, `UnsupportedMajor`, `UnsupportedMinor`, `UnsupportedRequiredFeature`, `BadEndianness`, `BadPageSize`, `ReservedNonZero`, and `ChecksumMismatch`.

## Test Matrix

| Area | Case | Expected Result |
| --- | --- | --- |
| encode | create default header | exact supported major/minor, page size `4096`, little-endian marker, zero reserved bytes, valid checksum |
| encode | create header with page size below `4096` | encode rejects before bytes are emitted |
| encode | create header with page size above `65536` | encode rejects before bytes are emitted |
| decode | decode bytes emitted by `encode()` | returns equivalent `Header` |
| decode | bootstrap shorter than `512` bytes | `FormatError` at public boundary |
| magic | unrelated file contents | `FormatError` |
| magic | recognizable but damaged `HEADER_MAGIC` | `Corruption` |
| version | unsupported major version | `IncompatibleVersion` |
| version | future unsupported minor version | `IncompatibleVersion` |
| features | unsupported required feature bit | `IncompatibleVersion` |
| features | unknown optional feature bit | accepted only if all required bits are supported |
| endianness | non-little-endian marker | `Corruption` |
| page size | page size below minimum | `Corruption` |
| page size | page size above maximum | `Corruption` |
| reserved | any required-zero reserved byte is non-zero | `Corruption` |
| checksum | flip one covered header byte | `Corruption` |
| checksum | flip one checksum byte | `Corruption` |
| checksum | changing reserved byte and recomputing checksum | `Corruption` from reserved validation |
| ordering | checksum field is ignored during checksum recomputation | valid encoded header still decodes |
| boundary | metadata is not consulted after header failure | open fails at header validation |
| I/O | read from offset `0` returns an OS error | `IoError` |

Tests should include byte-level assertions for stable offsets once M1 binds the exact layout. Corruption tests should mutate a valid encoded fixture so failures are deliberate and easy to diagnose.

## Dependencies And Handoff

- Depends on `format/constants.zig` defining header page id, bootstrap prefix size, page-size limits, version constants, magic bytes, and checksum algorithm selection.
- Depends on a CRC32C implementation or wrapper that can be shared later by metadata and normal-page codecs.
- Hands the decoded `page_size` to metadata validation and page I/O; no metadata page may be decoded before this succeeds.
- Hands immutable header fields to metadata validation so duplicated metadata fields can be compared against the header.
- Leaves selected metadata slot, root page validation, freelist validation, and file-length validation to the M1 open-path feature work.
- Gives M4 crash/corruption tests a deterministic first validation boundary before metadata candidate selection begins.

## Non-Goals

- Do not implement metadata A/B selection in the header codec.
- Do not validate root, freelist, branch, leaf, or overflow pages here.
- Do not repair, rewrite, or migrate a corrupted header during `open()`.
- Do not infer page size from file length or metadata.
- Do not add mutable runtime fields to the header.
- Do not add `mmap`, direct I/O, or page-cache behavior.
- Do not introduce new format constants that conflict with M0.

## Acceptance Checklist

- Header magic byte values are chosen once in M1 code and covered by tests.
- `create()` writes a `512`-byte bootstrap prefix with a valid CRC32C checksum.
- `open()` reads the bootstrap prefix before any page-sized structure.
- Header validation returns page size only after magic, version, features, endianness, page size, reserved fields, and checksum all pass.
- Unsupported major, future minor, and unsupported required feature bits map to `IncompatibleVersion`.
- Invalid magic shape, endianness, page size, reserved fields, or checksum map according to the error table above.
- Public errors do not leak inconsistent internal validation names.
- Metadata is never used to repair or override header validation failure.
- Tests cover success, field corruption, unsupported version/features, checksum behavior, and I/O boundary behavior.
- The header codec remains reusable by later metadata, recovery, and crash-test work without owning lifecycle or storage responsibilities.
