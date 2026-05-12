# Whole-Page Positional I/O

This plan defines the M1 storage I/O foundation for reading, writing, sizing, and syncing fixed-size database pages. It is intentionally narrow: it provides the safe page-addressing and durability primitives that header, metadata, root-page, freelist, and later commit code will use, without adding page-cache behavior, `mmap`, transactions, or B+Tree semantics.

## 1. Goals

- Provide one authoritative `page_offset(page_id, page_size)` calculation for all storage code.
- Read and write only complete database pages with positional I/O.
- Reject arithmetic overflow, truncated selected state, short reads, and short writes deterministically.
- Validate file length against selected metadata without treating unreachable tail bytes or tail pages as allocated or reusable.
- Provide sync wrappers that preserve M0 durability semantics while staying testable.
- Provide parent-directory sync behavior for strict database creation where the platform supports it.
- Keep the storage I/O layer independent from format codecs and metadata selection policy.

## 2. Proposed Module Boundary

Implement the feature in the storage boundary described by `docs/design/m1/m1-implementation-plan.md`, most likely under `src/storage/file.zig`.

The storage I/O module should own:

- raw file handle wrappers
- whole-page positional read and write helpers
- file length inspection
- complete-page length validation
- file flush wrappers
- parent-directory sync attempts
- test seams for short read, short write, and sync failure injection

The storage I/O module must not own:

- header, metadata, root-page, or freelist binary layout
- metadata A/B selection rules
- checksum computation
- transaction visibility or dirty-page ownership
- freelist interpretation
- public database lifecycle policy beyond returning precise storage errors

## 3. Core Types And Errors

M1 can keep the exact names internal, but the implementation should expose enough structure for `db.create()` and `db.open()` to make clear decisions.

Suggested internal types:

```zig
pub const PageId = u64;
pub const PageSize = u32;

pub const FileLengthInfo = struct {
    byte_len: u64,
    complete_pages: u64,
    trailing_bytes: u64,
};

pub const SyncMode = enum {
    strict,
    normal,
    unsafe,
};
```

Suggested storage-level error conditions:

- `IoError` for OS read, write, seek, flush, close, or sync failures.
- `ShortRead` internally, folded to public `IoError` unless open-path policy explicitly maps a selected critical-page truncation to `Corruption`.
- `ShortWrite` internally, folded to public `IoError`.
- `InvalidPageSize` internally, folded by callers according to context.
- `PageOffsetOverflow` internally, folded to public `Corruption` when caused by on-disk metadata and to `InvalidState` or `IoError` for impossible internal calls.
- `TruncatedFile` internally, folded to public `Corruption` when selected metadata requires pages beyond the complete file length.

The public boundary must stay aligned with M0: required I/O failures return `IoError`; selected metadata that claims unavailable complete pages returns `Corruption`.

## 4. `page_offset`

`page_offset(page_id, page_size) -> u64` is the only allowed way to convert a page id to a byte offset.

Rules:

- `page_size` must already be validated against the header and M0 bounds before normal use.
- `page_id` is zero-based.
- page `0` starts at offset `0`.
- page `1` starts at offset `page_size`.
- the function must check `page_id * page_size` for `u64` overflow.
- the function must not special-case metadata or data pages.
- the function must not accept a byte offset from callers as an alternative addressing mode.

Recommended implementation shape:

```zig
pub fn page_offset(page_id: PageId, page_size: PageSize) Error!u64 {
    return std.math.mul(u64, page_id, @as(u64, page_size)) catch error.PageOffsetOverflow;
}
```

Validation examples:

| Input | Result |
| --- | --- |
| `page_id = 0`, `page_size = 4096` | `0` |
| `page_id = 1`, `page_size = 4096` | `4096` |
| `page_id = 4`, `page_size = 4096` | `16384` |
| multiplication overflows `u64` | `PageOffsetOverflow` |

## 5. `read_exact_page`

`read_exact_page(file, page_id, buffer)` reads exactly one complete page from a page-aligned offset.

Rules:

- `buffer.len` must equal the validated `page_size`.
- the offset must be computed by `page_offset`.
- the operation must use positional I/O, not seek-then-read shared cursor state.
- the function must return success only after filling the entire buffer.
- EOF before `page_size` bytes is a short read.
- partial OS reads must be retried until the buffer is full or a terminal error/EOF occurs.
- the helper must not zero-fill missing bytes from a truncated file.
- the helper must not validate checksums or page types.

Open-path callers should perform file length validation before reading selected critical pages where possible. `read_exact_page` still must defend against races, external truncation, and malformed fixtures by detecting short reads.

Pseudo-flow:

```text
offset = page_offset(page_id, page_size)
filled = 0
while filled < page_size:
  n = pread(file, buffer[filled..], offset + filled)
  if n == 0:
    return ShortRead
  filled += n
return success
```

M1 should add a test seam that can force a short read after any byte count so the behavior is covered without relying on platform-specific filesystem races.

## 6. `write_exact_page`

`write_exact_page(file, page_id, buffer)` writes exactly one complete page to a page-aligned offset.

Rules:

- `buffer.len` must equal the validated `page_size`.
- the offset must be computed by `page_offset`.
- the operation must use positional I/O, not seek-then-write shared cursor state.
- the function must return success only after writing the entire page.
- partial OS writes must be retried until the page is complete or a terminal error occurs.
- a write returning zero bytes before completion is a short write.
- the helper must not compute checksums, infer page type, or mutate the buffer.
- the helper may grow the file when writing at the current tail, but metadata publication rules decide whether that new page becomes reachable.

Pseudo-flow:

```text
offset = page_offset(page_id, page_size)
written = 0
while written < page_size:
  n = pwrite(file, buffer[written..], offset + written)
  if n == 0:
    return ShortWrite
  written += n
return success
```

M1 should add a test seam that can force a short write and confirm the caller receives a storage I/O failure without silently publishing partial state.

## 7. File Length Validation

M1 needs two related checks: raw file length inspection and selected-state validation.

### 7.1 Inspect File Length

After validating the header page size, `open()` should inspect the file length and derive:

```text
complete_pages = byte_len / page_size
trailing_bytes = byte_len % page_size
```

Rules:

- `complete_pages` counts only whole pages.
- `trailing_bytes` are unreachable residue and never addressable as a page.
- partial trailing bytes do not make open fail by themselves.
- complete pages beyond selected metadata `page_count` do not make open fail by themselves.
- neither partial trailing bytes nor extra complete tail pages may be added to the freelist.

### 7.2 Validate Selected `page_count`

After metadata candidate validation has identified a selected state, validate:

```text
selected.page_count <= file_length.complete_pages
```

If this fails, the selected state references pages that are not fully present and open must return `Corruption`.

Additional page-id range checks are owned by metadata/root/freelist validation, but the storage helper can provide a reusable predicate:

```text
page_id < selected.page_count
```

### 7.3 Tail Handling Examples

| File State | Selected `page_count` | Result |
| --- | ---: | --- |
| exactly `5 * page_size` bytes | `5` | open may succeed |
| `4 * page_size` bytes | `5` | `Corruption` |
| `5 * page_size - 1` bytes | `5` | `Corruption` |
| `5 * page_size + 10` bytes | `5` | open may succeed; ignore trailing bytes |
| `7 * page_size` bytes | `5` | open may succeed; ignore pages `5..6` |
| `7 * page_size + 10` bytes | `5` | open may succeed; ignore pages `5..6` and trailing bytes |

This follows the M0 file growth rule: unreachable tail pages from failed commits are ignored on reopen.

## 8. Short Read And Short Write Policy

The storage layer must treat short I/O as explicit failure, not as a recoverable partial result.

Short read:

- occurs when `read_exact_page` cannot fill the requested page.
- returns an internal short-read error.
- maps to public `IoError` for direct helper failures.
- maps to public `Corruption` when open has already selected metadata that requires the missing page and the failure is best understood as truncated selected state.

Short write:

- occurs when `write_exact_page` cannot write the full page.
- returns an internal short-write error.
- maps to public `IoError`.
- must not be followed by in-memory metadata switching.
- must leave later recovery dependent only on previously synced metadata.

Tests should assert the exact boundary chosen by the M1 implementation so future M4 crash tests do not depend on incidental runtime behavior.

## 9. Sync Wrappers

M1 should provide explicit wrappers instead of scattering platform calls through lifecycle code.

Suggested functions:

```zig
pub fn sync_file(file: File, mode: SyncMode) Error!void
pub fn sync_parent_dir(path: []const u8, mode: SyncMode) Error!void
```

Rules:

- `strict` must issue the strongest supported flush required by M0 for the platform.
- `normal` may call the same implementation as `strict` in M1.
- `unsafe` may skip durable flushes, but M1 must not claim power-loss durability for it.
- sync failure must return `IoError`.
- callers must not publish newer in-memory metadata after a required sync fails.

Platform expectations from M0:

| Platform | `strict` file sync |
| --- | --- |
| Linux | `fsync()` |
| macOS | `fsync()` plus conservative `F_FULLFSYNC` path |
| Windows | `FlushFileBuffers()` |

M1 create-path usage:

1. write header, metadata A, invalid metadata B, empty root, and empty freelist pages.
2. call `sync_file(..., .strict)` or `.normal` aliasing strict.
3. call `sync_parent_dir(..., .strict)` where supported.
4. report create success only after required syncs complete.

M3 commit-path usage will later add the frozen two sync boundaries:

1. sync non-metadata pages before metadata publication.
2. sync the metadata page after writing the inactive slot.

## 10. Parent Directory Sync

Strict creation must flush the new directory entry where the platform supports that guarantee.

Rules:

- derive the parent directory from the database path.
- after file contents are flushed, open the parent directory with the minimal permissions needed to sync it.
- sync the parent directory before reporting strict create success.
- if the platform exposes a supported directory sync API and it fails, return `IoError`.
- if the platform does not support directory sync, document the fallback in code and tests; do not claim a stronger guarantee than the platform provides.
- directory sync must not be used to repair or validate database contents.

Recommended M1 stance:

- Linux and macOS should attempt parent directory sync.
- Windows should use the best supported equivalent available through Zig/std or platform APIs, and document any limitation if directory-handle flushing is not available.
- `normal` may alias `strict`.
- `unsafe` may skip parent directory sync, with no power-loss durability promise.

## 11. Test Matrix

The feature should add storage-focused tests that can be reused by the M1 open/create tests.

| Area | Case | Expected Result |
| --- | --- | --- |
| offset | `page_offset(0, 4096)` | returns `0` |
| offset | `page_offset(1, 4096)` | returns `4096` |
| offset | `page_offset(4, 4096)` | returns `16384` |
| offset | offset multiplication overflows `u64` | returns overflow error |
| read | read page `0` from a file with one complete page | fills full buffer |
| read | read page at a valid non-zero page id | uses positional offset, not shared cursor |
| read | read from a file shorter than the requested page | short read error |
| read | injected partial reads eventually fill the buffer | success |
| read | injected terminal short read before page completion | short read error |
| write | write page `0` to an empty file | file contains exactly the page bytes |
| write | write page at tail | file grows to include the complete page |
| write | write page at non-zero page id | writes at positional offset |
| write | injected partial writes eventually write full buffer | success |
| write | injected zero-byte write before completion | short write error |
| length | exact `page_count * page_size` length | complete pages equal `page_count`, no trailing bytes |
| length | partial trailing bytes | trailing bytes recorded as unreachable residue |
| length | selected `page_count` exceeds complete pages | `Corruption` at open boundary |
| length | complete tail pages beyond selected `page_count` | open may succeed; tail ignored |
| length | partial tail bytes beyond selected `page_count` | open may succeed; tail ignored |
| sync | strict file sync succeeds | create can continue |
| sync | strict file sync fails | `IoError`, create fails |
| sync | parent directory sync succeeds where supported | create can report success |
| sync | parent directory sync fails where required | `IoError`, create fails |
| sync | unsafe mode skips sync | no durability claim is made |

Integration tests should also cover:

- create writes five complete pages for the initial M1 file.
- reopen rejects selected metadata whose `page_count` exceeds complete file pages.
- reopen ignores complete or partial tail residue beyond selected `page_count`.
- short read of a selected root or freelist page fails clearly.
- short write during create prevents success.

## 12. Non-Goals

- Implementing `mmap`, direct I/O, async I/O, or a page cache.
- Reading or writing sub-page records.
- Inferring free pages from unreachable tail pages.
- Validating checksums, page types, or metadata generations inside the storage I/O layer.
- Implementing B+Tree mutation, freelist reuse, or overflow pages.
- Implementing M3 transaction commit ordering beyond providing sync wrappers.
- Implementing M4 crash-fault injection beyond deterministic short I/O and sync-failure seams.
- Providing full M7 cross-process locking.
- Repairing, truncating, or rewriting corrupted files during open.

## 13. Acceptance Checklist

This feature is complete when all of the following are true:

- all page reads and writes go through `page_offset`.
- `page_offset` checks multiplication overflow.
- `read_exact_page` uses positional I/O and fills exactly one page or fails.
- `write_exact_page` uses positional I/O and writes exactly one page or fails.
- page helpers reject buffers whose length differs from the validated page size.
- short reads and short writes are tested through deterministic seams.
- file length inspection records complete pages and trailing bytes separately.
- selected metadata with `page_count` beyond complete file pages is rejected.
- complete tail pages beyond selected `page_count` are ignored.
- partial trailing bytes beyond selected `page_count` are ignored.
- ignored tail pages are not inserted into the freelist.
- strict create flushes file contents before reporting success.
- strict create syncs the parent directory where the platform supports it.
- sync failures return `IoError` and do not publish newer in-memory state.
- `normal` is either implemented with documented lighter semantics or explicitly aliases `strict`.
- `unsafe` is documented as not providing power-loss durability.
- the storage module does not depend on metadata selection, checksum, or B+Tree code.
- tests cover offset calculation, exact reads, exact writes, file length validation, tail handling, sync failure, and parent directory sync behavior.
