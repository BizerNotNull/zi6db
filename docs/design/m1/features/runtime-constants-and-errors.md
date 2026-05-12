# Runtime Constants And Public Errors

## Goals

Implement the first M1 foundation slice: runtime constants and public error definitions. This work gives later M1 storage code one authoritative source for frozen M0 values and one stable public error boundary for lifecycle APIs.

The implementation must:

- encode the M0 format constants in Zig exactly once
- choose stable byte values for `HEADER_MAGIC` and `META_MAGIC`
- expose page ids, page sizes, format versions, page types, and checksum selection to format/storage modules without circular dependencies
- define the public `DBError` error set from `api-sketch.md`
- keep detailed validation failures internal while mapping public lifecycle calls to the documented M0 categories
- avoid implementing header, metadata, page I/O, locking, or transaction behavior in this feature beyond what is needed to compile constants and errors

## Related Modules And Files

Planned production files:

- `src/format/constants.zig`
  Owns all frozen runtime format constants.
- `src/errors.zig`
  Owns the public `DBError` set and shared internal-to-public mapping helpers.
- `src/format/validation.zig`
  Optional small helper module for internal validation result names if keeping them out of `errors.zig` is cleaner.
- `src/root.zig` or equivalent package entry file
  Re-exports public API-facing errors when the source layout is introduced.

Planned test files:

- `test/format_constants_test.zig` or module-adjacent tests in `src/format/constants.zig`
- `test/errors_test.zig` or module-adjacent tests in `src/errors.zig`

Design sources:

- `docs/design/m1/m1-implementation-plan.md`
- `docs/design/m0/format-constants.md`
- `docs/design/m0/api-sketch.md`
- `AGENTS.md`

## Implementation Steps

1. Create the source layout needed for this feature.
   Add `src/format/` if it does not exist, then add `constants.zig` and `errors.zig`. Keep the files independent: constants must not import database lifecycle code, storage code, or tests.

2. Define header and page constants.
   Add `HEADER_PAGE_ID = 0`, `META_A_PAGE_ID = 1`, `META_B_PAGE_ID = 2`, `FIRST_DATA_PAGE_ID = 3`, `HEADER_BOOTSTRAP_PREFIX_SIZE = 512`, `DEFAULT_PAGE_SIZE = 4096`, `MIN_PAGE_SIZE = 4096`, `MAX_PAGE_SIZE = 65536`, and `PAGE_ALIGNMENT` semantics as page-size alignment.

3. Define encoding constants.
   Add `FORMAT_MAJOR = 0`, `FORMAT_MINOR = 1`, a little-endian marker, and an enum or tag for `CRC32C`. Do not add alternate checksum algorithms in M1.

4. Choose and freeze magic byte values.
   Bind `HEADER_MAGIC` and `META_MAGIC` to explicit byte sequences. They must be fixed-width arrays, easy to identify in fixtures, and stable after the first M1-created file is emitted. Use distinct values so header and metadata pages cannot be confused by shape checks.

5. Define page type values.
   Add explicit numeric tags for `meta`, `freelist`, `branch`, `leaf`, and `overflow`. Reserve only what M0 requires; do not add future page types in this feature.

6. Add small validation helpers for constants.
   Implement helpers such as `isValidPageSize(page_size: u32) bool`, `isDataPageId(page_id: u64) bool`, and `pageOffset(page_id: u64, page_size: u32) !u64` only if they are constant-level arithmetic and do not require file I/O. Overflow from `pageOffset` should map through the public boundary as `Corruption` or `IoError` depending on the caller context; the helper may return an internal arithmetic error.

7. Define public errors.
   Add the `DBError` error set exactly as sketched by M0: `FormatError`, `IncompatibleVersion`, `ChecksumMismatch`, `Corruption`, `LockConflict`, `TransactionClosed`, `ReadOnlyTransaction`, `WriteConflict`, `InvalidState`, and `IoError`.

8. Define internal validation categories.
   Add a private or package-local internal error/result type for detailed validation failures, for example checksum mismatch, bad reserved field, unsupported version, invalid page size, slot mismatch, range violation, and short I/O shape. Keep these names out of the public API unless already included in `DBError`.

9. Add public mapping helpers.
   Implement a single mapping function used by future header, metadata, root, freelist, and storage code. The function should fold internal checksum and structure failures into the public categories described below.

10. Add tests for constant values and mappings.
    Tests should assert exact numeric values for all M0 constants, stable lengths/content for magic byte arrays, valid and invalid page-size boundaries, page type uniqueness, and internal-to-public error mapping.

## Data Structure And API Draft

```zig
pub const HEADER_PAGE_ID: u64 = 0;
pub const META_A_PAGE_ID: u64 = 1;
pub const META_B_PAGE_ID: u64 = 2;
pub const FIRST_DATA_PAGE_ID: u64 = 3;

pub const HEADER_BOOTSTRAP_PREFIX_SIZE: usize = 512;
pub const DEFAULT_PAGE_SIZE: u32 = 4096;
pub const MIN_PAGE_SIZE: u32 = 4096;
pub const MAX_PAGE_SIZE: u32 = 65536;

pub const FORMAT_MAJOR: u16 = 0;
pub const FORMAT_MINOR: u16 = 1;

pub const HEADER_MAGIC: [8]u8 = .{ 'z', 'i', '6', 'd', 'b', 'h', 'd', 'r' };
pub const META_MAGIC: [8]u8 = .{ 'z', 'i', '6', 'd', 'b', 'm', 'e', 't' };

pub const Endianness = enum(u8) {
    little = 1,
};

pub const ChecksumAlgorithm = enum(u8) {
    crc32c = 1,
};

pub const PageType = enum(u16) {
    meta = 1,
    freelist = 2,
    branch = 3,
    leaf = 4,
    overflow = 5,
};

pub fn isValidPageSize(page_size: u32) bool {
    return page_size >= MIN_PAGE_SIZE and
        page_size <= MAX_PAGE_SIZE and
        std.math.isPowerOfTwo(page_size);
}

pub fn isDataPageId(page_id: u64) bool {
    return page_id >= FIRST_DATA_PAGE_ID;
}
```

```zig
pub const DBError = error{
    FormatError,
    IncompatibleVersion,
    ChecksumMismatch,
    Corruption,
    LockConflict,
    TransactionClosed,
    ReadOnlyTransaction,
    WriteConflict,
    InvalidState,
    IoError,
};

pub const ValidationFailure = error{
    NotDatabaseBootstrap,
    BadMagic,
    BadEndianness,
    UnsupportedVersion,
    UnsupportedRequiredFeature,
    BadReservedField,
    BadChecksum,
    BadPageSize,
    BadPageType,
    BadSlotId,
    DuplicateFieldMismatch,
    PageIdOutOfRange,
    PageCountBeyondFile,
    EqualMetadataGeneration,
    NonMonotonicTxnId,
    ArithmeticOverflow,
};

pub fn publicErrorForValidation(err: ValidationFailure) DBError {
    return switch (err) {
        error.NotDatabaseBootstrap => error.FormatError,
        error.UnsupportedVersion,
        error.UnsupportedRequiredFeature,
            => error.IncompatibleVersion,
        error.BadChecksum => error.Corruption,
        else => error.Corruption,
    };
}
```

The exact Zig names may change to match the final package layout, but the values and mapping behavior should not drift from this plan.

## Error Mapping

| Condition | Public Error |
| --- | --- |
| Too few bytes or insufficient structure to identify a zi6db bootstrap | `FormatError` |
| Recognizable zi6db bootstrap with bad header magic or shape | `Corruption` |
| Bad endianness marker | `Corruption` |
| Required-zero reserved field is non-zero | `Corruption` |
| Header, metadata, root, or freelist checksum mismatch | `Corruption` |
| Unsupported `FORMAT_MAJOR` | `IncompatibleVersion` |
| Unsupported future `FORMAT_MINOR` | `IncompatibleVersion` |
| Unsupported required feature flag | `IncompatibleVersion` |
| Invalid page size below `MIN_PAGE_SIZE`, above `MAX_PAGE_SIZE`, or not power-of-two | `Corruption` |
| Metadata page size or format fields conflict with the validated header | `Corruption` |
| Metadata physical slot and encoded slot id disagree | `Corruption` |
| Metadata/header duplicated fields disagree | `Corruption` |
| Root or freelist page id is below `FIRST_DATA_PAGE_ID` or outside selected `page_count` | `Corruption` |
| Selected `page_count` exceeds complete whole pages in the file | `Corruption` |
| Metadata A/B candidates have equal generation | `Corruption` |
| Newer metadata generation has lower `txn_id` | `Corruption` |
| Required read, write, sync, create, close, or directory-sync operation fails | `IoError` |
| Requested lock guarantee cannot be provided | `LockConflict` |
| Double close or close with active future transactions | `InvalidState` |
| Future write operation attempted through read-only state | `ReadOnlyTransaction` |
| Future second writer attempted while a writer is active | `WriteConflict` |
| Future transaction handle reused after terminal state | `TransactionClosed` |

`ChecksumMismatch` may exist in `DBError` because it is part of the M0 sketch, but M1 lifecycle calls should fold checksum failures on critical structures into `Corruption` unless a lower-level diagnostic API is explicitly internal.

## Test Plan

Constants tests:

- assert all page ids match M0 values
- assert `FIRST_DATA_PAGE_ID` is greater than both metadata page ids
- assert `HEADER_BOOTSTRAP_PREFIX_SIZE` is `512`
- assert default/min/max page sizes are `4096`, `4096`, and `65536`
- assert `FORMAT_MAJOR = 0` and `FORMAT_MINOR = 1`
- assert `HEADER_MAGIC` and `META_MAGIC` are non-empty, fixed-width, distinct byte arrays
- assert page type tags are unique and stable
- assert checksum algorithm is `CRC32C`
- assert `isValidPageSize(4096)` and `isValidPageSize(65536)` are true
- assert `isValidPageSize(0)`, `isValidPageSize(2048)`, `isValidPageSize(65537)`, and non-power-of-two sizes are false

Error mapping tests:

- `NotDatabaseBootstrap` maps to `FormatError`
- `UnsupportedVersion` maps to `IncompatibleVersion`
- `UnsupportedRequiredFeature` maps to `IncompatibleVersion`
- `BadChecksum` maps to `Corruption`
- bad magic, bad endianness, reserved-field violation, slot mismatch, page range violation, equal metadata generation, and non-monotonic txn id map to `Corruption`
- storage-layer failures can be represented as `IoError`
- lock guard failures can be represented as `LockConflict`
- lifecycle misuse can be represented as `InvalidState`

Integration expectations for later M1 feature tests:

- header validation tests import constants instead of duplicating values
- metadata selection tests import page ids and page type tags instead of hard-coding them
- page I/O tests reuse page-size and page-offset helpers
- open/create lifecycle tests assert public errors, not internal validation names

## Dependencies And Handoff

This feature has no dependency on storage I/O, header encoding, metadata encoding, root/freelist page formats, or lock implementation. It should land before those features so they do not invent duplicate constants or local error sets.

Handoff requirements:

- Header codec work depends on `HEADER_MAGIC`, `HEADER_BOOTSTRAP_PREFIX_SIZE`, format versions, endianness, checksum algorithm, and page-size validation.
- Metadata codec work depends on `META_MAGIC`, metadata page ids, page type tags, format versions, required-feature mapping, and checksum-to-corruption mapping.
- Page I/O work depends on page-size bounds and shared page offset arithmetic.
- Open-path recovery depends on public error mapping being stable before validation rules are implemented.
- M3 transaction work depends on `TransactionClosed`, `ReadOnlyTransaction`, `WriteConflict`, and `InvalidState` already being reserved in the public error set.
- M7 locking work depends on `LockConflict` already being the public boundary for unsupported or conflicting lock guarantees.

## Non-Goals

- Do not implement header encode/decode.
- Do not implement metadata encode/decode or A/B selection.
- Do not implement root, branch, leaf, freelist, or overflow page codecs.
- Do not implement CRC32C itself unless a tiny enum/tag requires naming the algorithm.
- Do not implement file I/O, sync, directory sync, or lock guards.
- Do not implement `DB.open`, `DB.create`, `DB.close`, or transaction APIs.
- Do not change M0 constants, page numbering, checksum coverage, or public error semantics.
- Do not add performance optimizations, `mmap`, direct I/O, or cache behavior.

## Acceptance Checklist

- `src/format/constants.zig` is the only authoritative code location for M0 runtime constants.
- All M0 constants from `format-constants.md` are represented with explicit types.
- `HEADER_MAGIC` and `META_MAGIC` have stable, distinct byte values.
- Page type tags are explicit and unique.
- Page-size validation accepts only supported M0 page sizes.
- Public `DBError` contains every error listed in `api-sketch.md`.
- Internal validation failures map deterministically to public errors.
- Checksum validation failures on public lifecycle paths are documented to fold into `Corruption`.
- No storage, header, metadata, lock, transaction, or B+Tree behavior is implemented by this feature.
- Tests cover exact constants, boundary helpers, magic distinctness, page type uniqueness, and error mapping.
- Later M1 modules can import constants and errors without circular dependencies.
