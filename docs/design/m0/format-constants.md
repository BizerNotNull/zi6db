# M0 Format Constants

## Header And Page Constants

| Name | Value |
| --- | --- |
| `HEADER_PAGE_ID` | `0` |
| `META_A_PAGE_ID` | `1` |
| `META_B_PAGE_ID` | `2` |
| `FIRST_DATA_PAGE_ID` | `3` |
| `HEADER_BOOTSTRAP_PREFIX_SIZE` | `512` |
| `DEFAULT_PAGE_SIZE` | `4096` |
| `MIN_PAGE_SIZE` | `4096` |
| `MAX_PAGE_SIZE` | `65536` |
| `PAGE_ALIGNMENT` | equal to page size |

## Encoding Constants

| Name | Value |
| --- | --- |
| `ENDIANNESS` | `little-endian` |
| `CHECKSUM_ALGORITHM` | `CRC32C` |
| `FORMAT_MAJOR` | `0` |
| `FORMAT_MINOR` | `1` |

## Magic Values

M0 freezes symbolic names and semantics. The exact byte sequences may be bound in M1 code, but they must remain stable once the format is emitted.

| Name | Meaning |
| --- | --- |
| `HEADER_MAGIC` | identifies a zi6db file header |
| `META_MAGIC` | identifies a metadata page |

## Page Types

| Name | Meaning |
| --- | --- |
| `PAGE_TYPE_META` | metadata page |
| `PAGE_TYPE_FREELIST` | freelist page |
| `PAGE_TYPE_BRANCH` | B+Tree internal page |
| `PAGE_TYPE_LEAF` | B+Tree leaf page |
| `PAGE_TYPE_OVERFLOW` | overflow/value spill page |

## Reserved Invariants

- all page ids are unsigned 64-bit logical identifiers
- all on-disk integer fields use little-endian encoding
- the page checksum field is excluded from its own checksum coverage
- metadata and normal-page checksums are mandatory in the first format
