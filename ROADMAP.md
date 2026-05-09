# zi6db Roadmap

`zi6db` aims to be a **100% embedded, single-file KV database for Zig** inspired by the simplicity of `bbolt` and the safety/performance goals of `redb`.

The end goal is not "a file that can store bytes", but a production-usable embedded database with strong correctness guarantees, predictable performance, and a small API surface.

## Final Product Goals

At the end of the roadmap, `zi6db` should provide:

### 1. Embedded and Single-File

- No server process, no background daemon, no network dependency.
- One database = one file on disk.
- Open/create/close database files from a Zig application.
- Cross-platform behavior for Linux, macOS, and Windows.

### 2. Durable ACID Transactions

- Read-only transactions.
- Read-write transactions.
- Atomic commit and rollback.
- Crash-safe commits.
- Consistent snapshots for readers.
- Optional explicit sync policy for durability/performance tradeoffs.

### 3. Concurrent Access Model

- Multiple concurrent readers.
- Single writer semantics.
- File locking to prevent database corruption from multi-process misuse.
- Clear transaction lifetime rules.

### 4. Ordered KV Storage Engine

- Ordered key-value storage.
- `get`, `put`, `delete`, `exists`.
- Lexicographic key ordering.
- Efficient point lookups.
- Efficient forward and backward iteration.
- Range scan and prefix scan support.

### 5. Namespaces / Logical Collections

- Bucket or table abstraction similar to `bbolt` / `redb`.
- Create/drop/list buckets or tables.
- Isolated key spaces per bucket/table.
- Optional nested buckets if we choose the `bbolt`-style model.

### 6. Cursor / Iterator API

- Seek to first/last item.
- Seek to exact key or nearest key.
- Move next/prev.
- Iterate over a whole bucket/table.
- Iterate over key ranges without materializing all results.

### 7. On-Disk Storage Management

- Page-based file format.
- Metadata pages with format/version information.
- B+Tree or equivalent ordered page structure.
- Overflow page handling for large values.
- Free page tracking and space reuse.
- Stable on-disk format with upgrade/version checks.

### 8. Integrity and Recovery

- Checksum or equivalent corruption detection for critical metadata.
- Startup validation of file headers and metadata.
- Detect incomplete or invalid commits after crashes.
- Read-only integrity check / verify tool or API.

### 9. Operational Features

- Database statistics API.
- File size / page usage / free space reporting.
- Consistent backup or snapshot export.
- Optional compaction / vacuum to rewrite and shrink the file.

### 10. Developer Experience

- Small, idiomatic Zig API.
- Zero-copy or low-copy reads where safe.
- Good error types and actionable failure messages.
- Strong test coverage, especially for crash safety and corruption cases.
- Benchmarks against representative workloads.

## Suggested Scope Decisions

To keep the project focused, these should be part of the intended shape:

- Keys and values are raw bytes.
- Data is ordered by key.
- The database is optimized for local embedded usage, not remote access.
- The API should prefer correctness and simplicity over too many tuning knobs.

## Non-Goals

These are explicitly out of scope for the core database:

- SQL layer.
- Distributed replication.
- Network protocol / client-server mode.
- Full-text search engine behavior.
- Multi-writer concurrent commits.