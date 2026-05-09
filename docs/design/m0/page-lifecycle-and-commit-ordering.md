# M0 Page Lifecycle And Commit Ordering

This document freezes page allocation, dirty-page ownership, freelist transitions, and crash-safe commit ordering.

## 1. Page Lifecycle

Under copy-on-write, a page moves through these logical states:

`visible clean -> cloned/new -> dirty -> written -> synced -> committed`

Or, on abort:

`visible clean -> cloned/new -> dirty -> discarded`

Visible pages are never modified in place.

## 2. Allocation Sources

A write transaction allocates pages from:

1. reusable freelist pages
2. file-tail extension

Pages that the same transaction deletes do not immediately return to reusable state.

## 3. Freelist Semantics

The freelist has two persisted sets:

- `reusable`
- `pending_free[freeing_txn_id]`

### 3.1 Delete Rule

When a write transaction removes the last reference to a page:

- the page is appended to `pending_free[current_txn_id]`
- it is not added to `reusable` yet

### 3.2 Reuse Rule

A pending-free page may move into `reusable` only when every read transaction older than `freeing_txn_id` has ended.

### 3.3 Restart Rule

In M1 single-process mode, after restart there are no surviving readers. Therefore, pending-free groups from the selected freelist may be migrated to reusable if their generational rule is satisfied by the newly opened process state.

### 3.4 Safety Rule

The current writer must not reuse pages it just freed if those pages may still be reachable from an older reader snapshot.

## 4. Dirty-Page Ownership

Only the active write transaction may own dirty pages.

It owns:

- cloned pages derived from the visible tree
- newly allocated branch/leaf/overflow pages
- the new freelist page
- the inactive metadata page contents before writeout

Read transactions only read committed pages reachable from their pinned metadata snapshot.

## 5. Commit Algorithm

The commit path is frozen to the following sequence:

1. Acquire the writer commit path exclusively.
2. Allocate `new_txn_id = previous_txn_id + 1`.
3. Allocate `new_generation = previous_generation + 1`.
4. Finalize all dirty branch, leaf, and overflow pages with checksums.
5. Finalize the new freelist page with:
   - updated reusable set
   - updated `pending_free`
6. Write all non-metadata pages to page ids that are not part of the currently visible state.
7. Perform the required pre-metadata file sync.
8. Build the inactive metadata page with:
   - new root page id
   - new freelist page id
   - `new_txn_id`
   - `new_generation`
   - new `page_count`
9. Compute the metadata checksum.
10. Write the inactive metadata page to the opposite metadata slot.
11. Perform the required metadata sync.
12. Switch the current in-memory metadata pointer.
13. Release the writer lock.
14. Mark the transaction closed.

Any failure before step 12 leaves in-memory current metadata unchanged.

## 6. Sync Boundaries

M0 freezes two durability boundaries for `strict` mode:

1. after all non-metadata pages are written
2. after the metadata page is written

This ensures that recovery never selects metadata that references pages which were never durably written first, within the promised failure model.

## 7. Platform Rules

### 7.1 Linux

- `strict` uses `fsync()`
- `normal` may use `fdatasync()` if implementation evidence shows metadata required for file growth is still preserved correctly
- `unsafe` may skip one or both syncs

### 7.2 macOS

- `strict` uses `fsync()` plus the conservative `fcntl(F_FULLFSYNC)` path
- `normal` may use `fsync()` without `F_FULLFSYNC`
- `unsafe` may skip durable flushes

### 7.3 Windows

- `strict` uses `FlushFileBuffers()`
- `normal` may use the same implementation initially
- `unsafe` may omit buffer flushes

### 7.4 New File Creation

When creating a new database file in `strict` mode:

- the file contents must be flushed
- the parent directory entry must also be synced where the platform supports that guarantee

## 8. File Growth Rules

- `page_count` is the exclusive upper bound of allocated page ids.
- Tail growth reserves new page ids before metadata publication.
- If extending the file or writing a tail page fails, `commit()` fails.
- Unreachable tail pages from failed commits are ignored on reopen.

## 9. Locking Boundary

M0 freezes these boundaries:

- M1 guarantees single-process safety for multiple readers and one writer
- cross-process write access may be rejected outright before M7
- M7 may add shared/exclusive file locking, but it may not weaken M0 snapshot or commit guarantees

## 10. Crash Outcome Rules

- Crash before non-metadata sync: old metadata must remain selectable
- Crash after non-metadata sync but before metadata write: old metadata must remain selectable
- Crash during metadata write: recovery selects whichever metadata page validates as newest
- Crash after metadata write but before metadata sync: recovery may still select new metadata if it validates
- Crash after metadata sync: recovery must select the new metadata

## 11. Durability Modes

M0 freezes API semantics for three durability labels:

- `strict`
  - promises power-loss-safe commit after `commit()` success
  - requires both sync boundaries
- `normal`
  - reserves a lighter implementation, but still promises that successful commit is crash-recoverable
  - M1 may implement it as an alias of `strict`
- `unsafe`
  - may skip some or all syncs
  - does not promise power-loss durability
  - is only required to avoid exposing in-process partial state before commit returns

## 12. Failure Terminal State

If `fsync`, `fdatasync`, `F_FULLFSYNC`, or `FlushFileBuffers` fails:

- `commit()` returns `IoError`
- the write transaction becomes terminal and unusable
- the caller may only discard it
- the database must not switch current metadata in memory

The caller must assume the new metadata may or may not already be durable unless metadata sync success was confirmed.
