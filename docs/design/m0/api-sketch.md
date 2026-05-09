# M0 API Sketch

This document freezes the developer-facing lifecycle and error semantics needed for M1-M5 without overcommitting the future data model.

## 1. Open And Create

```zig
pub const OpenOptions = struct {
    create_if_missing: bool = false,
    read_only: bool = false,
    page_size: u32 = 4096,
    durability_mode: DurabilityMode = .strict,
    lock_mode: LockMode = .single_process,
    max_read_transactions: u32 = 1024,
};

pub const DurabilityMode = enum {
    strict,
    normal,
    unsafe,
};

pub const LockMode = enum {
    single_process,
    multi_process,
};

pub fn open(path: []const u8, options: OpenOptions) !DB;
pub fn create(path: []const u8, options: OpenOptions) !DB;
pub fn close(db: *DB) !void;
```

### 1.1 Open Rules

- `open()` validates header, metadata, and selected root/freelist pages.
- `open()` returns `IncompatibleVersion` for unsupported format versions or required feature flags.
- `open()` returns `Corruption` for checksum failures, self-inconsistent metadata, or invalid selected critical pages.
- `open()` in `read_only` mode must reject any operation that would start a write transaction.

### 1.2 Create Rules

- `create()` initializes header, metadata A, empty root leaf, and empty freelist.
- `create()` must not create two valid metadata slots with the same generation.
- `create_if_missing` may be supported through `open()` later, but the on-disk initialization sequence remains the same.

## 2. Transaction API

```zig
pub fn beginRead(db: *DB) !ReadTx;
pub fn beginWrite(db: *DB) !WriteTx;

pub fn id(tx: *const Tx) u64;
pub fn commit(tx: *WriteTx) !void;
pub fn rollback(tx: *Tx) void;
pub fn isReadOnly(tx: *const Tx) bool;
```

### 2.1 Read Transaction Rules

- `beginRead()` pins the current metadata snapshot.
- a read transaction remains valid until rollback or close
- a read transaction cannot be upgraded implicitly to a write transaction

### 2.2 Write Transaction Rules

- at most one write transaction may exist at a time
- a write transaction may coexist with read transactions
- after `commit()` success, the transaction is closed
- after `commit()` failure, the transaction is terminal and must not be reused
- after `rollback()`, the transaction is closed

## 3. Bucket And Cursor Placeholders

M0 only freezes the lifecycle shape:

```zig
pub fn getBucket(tx: *Tx, name: []const u8) !?Bucket;
pub fn createBucket(tx: *WriteTx, name: []const u8) !Bucket;
pub fn deleteBucket(tx: *WriteTx, name: []const u8) !void;

pub fn first(cursor: *Cursor) !void;
pub fn last(cursor: *Cursor) !void;
pub fn seek(cursor: *Cursor, key: []const u8) !void;
pub fn next(cursor: *Cursor) !void;
pub fn prev(cursor: *Cursor) !void;
pub fn key(cursor: *const Cursor) ?[]const u8;
pub fn value(cursor: *const Cursor) ?[]const u8;
```

Buckets and cursors may evolve internally in later milestones, but the following semantics are frozen now:

- cursor lifetime is bound to its transaction
- closing or rolling back the transaction invalidates all cursors
- a cursor from one transaction must never observe another transaction's uncommitted data

## 4. Error Model

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
```

### 4.1 Error Meanings

- `FormatError`
  - structurally invalid input that is not recognized as a supported database layout
- `IncompatibleVersion`
  - valid zi6db file, but requires a newer major/minor format or unsupported required feature
- `ChecksumMismatch`
  - internal validation result that may be folded into `Corruption` at the public API boundary
- `Corruption`
  - the file is recognized, but required structures are invalid or inconsistent
- `LockConflict`
  - lock mode or external lock ownership prevents the requested open or transaction
- `TransactionClosed`
  - the caller reused a committed, rolled-back, or failed transaction
- `ReadOnlyTransaction`
  - attempted a mutating operation in a read-only transaction or database
- `WriteConflict`
  - no second writer may start while one is active
- `InvalidState`
  - API misuse that violates lifecycle constraints
- `IoError`
  - OS or storage layer error during read, write, sync, create, or close

## 5. `close()` Policy

M0 freezes the conservative rule:

- `close()` must fail with `InvalidState` if there are active transactions
- the caller must end read and write transactions explicitly before closing the database

This avoids hidden transaction cancellation behavior in early milestones.

## 6. Durability Contract At API Level

At the API boundary:

- `commit()` success means the durability guarantee of the selected mode has been met
- `commit()` failure before metadata sync confirmation does not prove the commit was absent from disk
- after ambiguous failure, reopening and recovery are the authoritative source of truth

## 7. Non-Goals Of The API Sketch

M0 intentionally does not freeze:

- nested bucket semantics
- duplicate-key semantics
- custom comparators
- compaction APIs
- backup/export APIs

Those may be added later as long as they respect the storage and transaction rules frozen by M0.
