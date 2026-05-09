# M0 Transaction And Recovery

This document freezes transaction visibility, metadata switching, rollback semantics, and deterministic startup recovery.

## 1. Transaction Model

`zi6db` uses:

- multiple concurrent read transactions inside one process
- at most one write transaction at a time
- copy-on-write for all pages that can become visible to readers

Read transactions pin a metadata snapshot selected at begin time. They never observe later commits.

Write transactions:

- own all newly allocated pages
- clone any visible page before mutation
- stage dirty pages privately until commit
- never overwrite the visible root, freelist, branch, leaf, or overflow page in place

## 2. Snapshot Semantics

### 2.1 Read Transactions

When `beginRead()` starts:

1. It reads the currently selected in-memory metadata snapshot.
2. It records that snapshot's `generation` and `txn_id`.
3. It traverses only pages reachable from that root and freelist.

The read transaction keeps that snapshot until rollback/close. It does not rebase to newer metadata.

### 2.2 Write Transactions

When `beginWrite()` starts:

1. It captures the current metadata snapshot as its base.
2. It becomes the only active writer.
3. It may run concurrently with read transactions that still use the older root.

The writer produces:

- new dirty branch/leaf/overflow pages
- a new freelist page
- a new root page when the tree shape changes
- a new metadata page in the inactive metadata slot

## 3. Metadata Slot Switching

The selected metadata alternates strictly between slots A and B.

- If the current selected slot is A, the next commit must write B.
- If the current selected slot is B, the next commit must write A.
- New database initialization writes only slot A as valid with `generation = 1`.
- Slot B must be zeroed or otherwise invalid on first creation.

M0 forbids starting with two valid metadata pages that share the same generation.

## 4. Commit Success And Failure Semantics

The commit protocol itself is described in [page-lifecycle-and-commit-ordering.md](./page-lifecycle-and-commit-ordering.md). This section freezes the externally visible semantics.

### 4.1 Success Point

`commit()` may return success only after:

1. all non-metadata pages for the commit were written
2. the required pre-metadata data sync succeeded
3. the inactive metadata page was written
4. the required metadata sync succeeded

Only after that may the process switch the current metadata in memory.

### 4.2 Failure Classes

#### Class A: failure before metadata write begins

- `commit()` returns `IoError`
- current in-memory metadata does not switch
- the write transaction enters a terminal failed state
- recovery must select the previous valid metadata

#### Class B: metadata write may have started, but metadata sync is not confirmed

- `commit()` returns `IoError`
- current in-memory metadata does not switch
- the write transaction enters a terminal failed state
- recovery may select either the old metadata or the new metadata
- the caller must treat recovery as the only authoritative answer

#### Class C: metadata sync confirmed

- `commit()` may return success
- recovery must be able to select this new metadata within the promised failure model

## 5. Rollback Semantics

Rollback affects only in-memory transaction state.

Rollback discards:

- dirty page clones
- newly allocated but not yet committed page ownership
- in-memory pending-free updates
- staged root and freelist changes

Rollback does not repair disk state because no visible page was overwritten in place.

## 6. Startup Recovery Algorithm

Startup recovery is deterministic and must follow this order:

1. Read the fixed `512`-byte header bootstrap prefix.
2. Validate header magic, version, endianness, page size, and header checksum.
3. Decode `page_size` from the header.
4. Read metadata A and metadata B using that page size.
5. Validate each metadata page:
   - metadata magic
   - page type = `META`
   - page size matches header
   - format version compatibility
   - checksum
   - `root_page_id`, `freelist_page_id`, and `page_count` ranges
   - referenced root page type and checksum
   - referenced freelist page type and checksum
6. Discard invalid metadata pages.
7. If both metadata pages are invalid, return `Corruption`.
8. If exactly one is valid, select it.
9. If both are valid, compare `generation`.
10. Select the metadata with the larger `generation`.
11. If generations are equal, return `Corruption`.
12. Publish the selected metadata as the opened database state.

## 7. Equal-Generation Rule

If metadata A and B have the same `generation`:

- if either differs in content: return `Corruption`
- if every field is identical: still return `Corruption`

This conservative rule prevents the engine from masking initialization bugs, duplicate-write bugs, or broken slot-switch logic.

## 8. `generation` And `txn_id`

M0 freezes these roles:

- `generation`
  - startup recovery ordering field
  - commit sequence identifier for visible states
- `txn_id`
  - transaction visibility marker
  - freelist pending/reusable cutoff input
  - debugging and observability aid

On every committed write transaction:

- `generation = previous_generation + 1`
- `txn_id = previous_txn_id + 1`

If recovery finds monotonicity broken, it returns `Corruption`.

Root and freelist validation are part of metadata validity, not a later tie-breaker. A metadata page that points to an out-of-range page, the wrong page type, or a page with a checksum failure is invalid before slot selection.

## 9. Recovery Scope

Recovery only tries to identify the last valid committed state. It does not:

- salvage partially written tree pages
- guess intended page reachability
- reclaim orphaned tail pages implicitly
- repair a corrupted header from metadata

Pages not reachable from selected metadata are ignored for correctness purposes.

## 10. Reopen Guarantees

After a crash or power loss:

- if only old metadata validates, the old state is selected
- if new metadata validates and is newer, the new state is selected
- if neither validates, open fails with `Corruption`

This is the only authoritative persisted state decision.
