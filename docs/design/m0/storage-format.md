# M0 Storage Format

This document freezes the on-disk format that M1-M6 must follow unless a future format-version upgrade explicitly replaces it.

## 1. File Layout

`zi6db` is a single file split into fixed-size pages.

- `page 0`: immutable file header
- `page 1`: metadata slot A
- `page 2`: metadata slot B
- `page >= 3`: normal data pages

The default page size is `4096` bytes. All page I/O is positional read/write against whole-page offsets.

```text
+-----------+-----------+-----------+-------------------+
| page 0    | page 1    | page 2    | page 3..N         |
| header    | meta A    | meta B    | normal pages      |
+-----------+-----------+-----------+-------------------+
```

## 2. Frozen Constants

The complete constant table lives in [format-constants.md](./format-constants.md). The first format freezes these values:

- Endianness: little-endian
- Default page size: `4096`
- Minimum page size: `4096`
- Maximum page size: `65536`
- Header bootstrap prefix size: `512` bytes
- Fixed page ids:
  - `0`: header
  - `1`: metadata A
  - `2`: metadata B
  - `3`: first data page
- Checksum algorithm: `CRC32C`

## 3. Header Rules

The file header is an immutable bootstrap record. Runtime state must not be rewritten into it after database creation.

### 3.1 Header Responsibilities

The header stores only:

- file magic
- format major/minor version
- endianness marker
- page size
- feature flags
- header checksum

The header does not store:

- current root page
- current freelist page
- current transaction id
- current generation
- current page count

### 3.2 Header Validation

Open must read the first `512` bytes before interpreting any page-sized structures.

If any of the following checks fail, `open()` returns `Corruption`:

- header magic mismatch
- unsupported endianness marker
- `page_size` outside the frozen min/max range
- header checksum mismatch
- reserved bits that are required to be zero are non-zero

Metadata is not allowed to repair a corrupted header. If a header field duplicates a metadata field and the values disagree, `open()` returns `Corruption`.

## 4. Metadata Page Format

Pages `1` and `2` are dedicated metadata slots. Exactly one valid metadata page defines the current visible database state.

### 4.1 Metadata Responsibilities

Each metadata page stores:

- metadata magic
- page type = `META`
- format major/minor version
- page size
- slot id (`A` or `B`)
- `generation`
- `txn_id`
- `root_page_id`
- `freelist_page_id`
- `page_count`
- state flags
- metadata checksum

### 4.2 Metadata Selection Fields

- `generation` is the only primary ordering field for startup recovery.
- `txn_id` is an auxiliary consistency and visibility field.
- Under normal commits both increase monotonically together.
- If `generation` is newer but `txn_id` goes backwards, recovery returns `Corruption`.

### 4.3 Metadata Invariants

- `root_page_id` and `freelist_page_id` must be `>= 3`
- `root_page_id` and `freelist_page_id` must be `< page_count`
- `page_count` is the exclusive upper bound of allocated page ids
- `page_count >= 4`
- the checksum covers the full metadata page except the checksum field itself

## 5. Common Normal-Page Header

Every page except the file header shares a common prefix:

- `page_id`
- `page_type`
- `page_generation`
- flags
- checksum

Checksums for normal pages cover:

`checksum(page_id || page_type || page_generation || flags || page_body_without_checksum)`

Validation failure makes the page invalid. The engine does not attempt in-place repair.

## 6. Page Types

The following page types are reserved in M0:

- `META`
- `FREELIST`
- `BRANCH`
- `LEAF`
- `OVERFLOW`

`OVERFLOW` is format-reserved even if full large-value support lands later. M6 must use this reserved type instead of redefining the disk format.

## 7. B+Tree Page Structure

### 7.1 Branch Pages

Branch pages store:

- common page header
- key count
- rightmost child pointer
- slot directory
- variable-length key payload area

Each branch slot stores:

- child page id for the subtree left of the key
- key offset
- key length

Keys are stored in sorted lexicographic byte order.

### 7.2 Leaf Pages

Leaf pages store:

- common page header
- entry count
- optional right-sibling page id reserved field
- slot directory
- variable-length key/value payload area

Each leaf slot stores:

- key offset
- key length
- value offset
- value length
- flags

Value flags reserve space for later overflow references without changing the slot layout.

## 8. Freelist Page Structure

The freelist page stores both reusable and pending-free state.

It contains:

- common page header
- reusable page id list
- `pending_free` groups keyed by `freeing_txn_id`
- reserved room for compaction or chunked freelist encoding in later milestones

M0 freezes that `pending_free` is persisted on disk. Restart recovery must consult the freelist page selected by current metadata instead of inferring free space from unreachable file pages.

## 9. Overflow Reservation

Overflow pages are reserved with this semantic contract:

- page type is `OVERFLOW`
- an overflow chain belongs to exactly one logical value version
- overflow pages are copy-on-write and are never overwritten in place
- branch and leaf payloads must be able to encode an inline value or an overflow reference

The first implementation may reject oversized values, but it must preserve the on-disk hooks above.

## 10. File Growth

- New pages are allocated from the current freelist reusable set or from the file tail.
- Tail allocation consumes the next page id below `page_count`, then increases `page_count`.
- If file growth fails before metadata is written, the commit fails and the old metadata remains authoritative.
- Extra tail pages from failed commits are ignored on the next open if they are not reachable from selected metadata.

## 11. Compatibility Policy

- `format_major` mismatch: return `IncompatibleVersion`
- `format_minor` greater than supported minor: return `IncompatibleVersion`
- recognized but unsupported required feature flag: return `IncompatibleVersion`
- corruption in an otherwise supported version: return `Corruption`

Future milestones may add optional features only through reserved flags or a later format-version bump.

## Appendix: Format Notes

- The first version freezes positional I/O as the only supported storage path.
- `mmap` may be added later only as an implementation detail that preserves this format, commit ordering, and recovery behavior.
- The header is immutable after creation.
- Metadata is the only on-disk commit switch point.
