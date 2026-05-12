# Freelist Promotion And Page Reuse

This feature plan makes the M6 freelist behavior concrete for reusable-page
promotion, page allocation, and safe reuse across transaction boundaries. It is
derived from `docs/design/m6/m6-implementation-plan.md`,
`docs/design/m3/features/rollback-cleanup-and-freelist-reader-safety.md`, and
the repository rules in `AGENTS.md`.

The goal is to finish M6 with a production-ready page reuse model that:

- persists both `reusable` and `pending_free` state through the normal commit
  path
- promotes pending groups only when the oldest active reader cutoff proves reuse
  is safe
- never reuses pages freed by the active writer inside the same transaction
- allocates from reusable space before growing the logical file tail
- leaves explicit seams for M7 verify, compaction, and cross-process reader
  tracking

## 1. Scope

In scope:

- persisted and in-memory `reusable` and `pending_free` models
- promotion from pending groups into reusable space
- `oldest_reader_txn_id` cutoff rules and when they apply
- allocation order for tree, overflow, and freelist pages
- same-transaction no-reuse rules
- tests for reuse correctness, reader safety, restart behavior, and crash
  boundaries
- M7 handoff seams for verification and future compaction

Out of scope:

- cross-process reader lock tables
- background reclamation or asynchronous promotion
- file truncation, vacuum, or compaction as part of normal delete/update paths
- rebuilding free space by scanning unreachable file pages
- changing M0 page layout, metadata switching, or recovery rules

## 2. Core Intent

M6 should treat reuse as a commit-published lifecycle with two durable free-page
states:

- `reusable`: page IDs that a later writer may allocate immediately
- `pending_free[freeing_txn_id]`: page IDs that are logically free but still
  blocked by possible older readers

Every freed page follows this progression:

```text
live page -> pending_free[freeing_txn_id] -> reusable -> allocated again
```

Two safety rules define the feature:

1. A write transaction publishes newly freed pages only as
   `pending_free[current_txn_id]`.
2. A page may move from `pending_free[X]` to `reusable` only when no active
   reader has `snapshot_txn_id < X`.

This keeps M6 aligned with the M0 metadata-switch model and preserves the M3
reader-safety semantics without introducing special cases for overflow pages or
freelist pages.

## 3. Reusable And Pending Models

The in-memory writer view should distinguish four sets:

- `base_reusable`: reusable ranges or page IDs decoded from the pinned freelist
  snapshot
- `base_pending`: committed pending groups keyed by `freeing_txn_id`
- `allocated_in_tx`: pages allocated by the active writer
- `freed_in_tx`: pages newly freed by the active writer and destined for
  `pending_free[new_txn_id]`

Recommended model:

- store `reusable` as sorted page ranges with exact page-count accounting
- store `pending_free` as sorted groups keyed by `freeing_txn_id`
- keep `allocated_in_tx` and `freed_in_tx` transaction-local and never visible
  outside the active writer
- treat old tree pages, old overflow pages, and old freelist pages as the same
  freed-page class once they enter `pending_free`

Required invariants:

- a page ID appears in at most one of `reusable`, `pending_free`,
  `allocated_in_tx`, and `freed_in_tx`
- fixed pages `0`, `1`, and `2` never appear in any free-page structure
- the persisted freelist image rebuilds the exact reusable/pending state during
  `open()`
- recovery never infers free pages from unreferenced tree pages or ignored tail
  pages

## 4. Oldest Reader Cutoff

M6 should inherit the M3 cutoff rule without weakening it:

- a pending group with `freeing_txn_id = X` is promotable when no active reader
  has `snapshot_txn_id < X`
- equivalently, if `oldest_reader_txn_id` exists, group `X` is promotable when
  `oldest_reader_txn_id >= X`
- if there are no active readers, all committed pending groups from the pinned
  freelist snapshot are promotable

Why equality is safe:

- a reader at snapshot `X` started after transaction `X` committed
- that reader cannot still reach the pre-delete page versions from before `X`
- only readers with snapshot IDs lower than `X` can still observe those pages

Promotion timing:

- promotion may run when a writer builds its allocator view at `beginWrite()`
- promotion may run again during commit finalization before the new freelist
  image is serialized
- promotion becomes durable only when the new freelist image is committed and
  selected through metadata publication

Required rules:

- never promote the active writer's own `freed_in_tx`
- never mutate the pinned base freelist snapshot in place
- after restart in the M6 single-process model, previously persisted pending
  groups are eligible for promotion because no in-process readers survive
- even after restart, promotion still becomes durable only through a later
  committed freelist rewrite

## 5. Same-Transaction No Reuse

Pages freed by the active writer must not be allocated again by that same write
transaction.

This rule applies equally to:

- replaced branch pages
- replaced leaf pages
- deleted overflow chain pages
- old freelist pages replaced by a new freelist image

Implementation rules:

- all pages freed by the active writer accumulate only in `freed_in_tx`
- commit appends `freed_in_tx` as `pending_free[new_txn_id]`
- the allocator never draws from `freed_in_tx`, even if the writer later needs
  additional tree, overflow, or freelist pages
- rollback drops `freed_in_tx` and all private allocations without publishing
  any reuse state

Reason for the rule:

- an older reader may still follow the pre-commit root and reach the old page
  contents
- reusing the same page ID inside the freeing transaction could let that reader
  observe unrelated bytes through a pinned snapshot

## 6. Allocation Order

The allocator should use one shared policy for tree pages, overflow pages, and
freelist pages:

1. Build a transaction-local reusable view by promoting eligible committed
   pending groups.
2. Allocate from reusable ranges when a suitable run exists.
3. Allocate individual reusable pages when no contiguous run is available.
4. Allocate from the logical file tail by advancing the transaction-local
   `page_count`.

Required allocation rules:

- allocation from reusable removes the returned page IDs immediately
- the same page ID must never be returned twice in one transaction
- blocked pending groups remain unavailable
- `freed_in_tx` remains unavailable
- tail allocation returns only page IDs greater than or equal to the writer's
  starting `page_count`
- correctness must not depend on contiguous reuse, even if contiguous runs are
  preferred for overflow chains

Practical policy choices for M6:

- prefer exact or best-fit runs for overflow chains when reusable ranges allow
- fall back to fragmented reusable pages before growing the tail
- grow the tail only after reusable space is exhausted for the requested
  allocation

## 7. Commit And Restart Integration

The writer should build the next freelist image from:

- base reusable pages from the pinned snapshot
- eligible base pending groups promoted into the writer's reusable view
- blocked base pending groups preserved under their original
  `freeing_txn_id`
- reusable pages consumed by this transaction removed from the reusable set
- pages newly freed by this transaction appended as `pending_free[new_txn_id]`
- new freelist pages excluded from the free-page structures they serialize

Commit rules:

- write the new freelist page or freelist chain before metadata publication
- free the old freelist pages into `pending_free[new_txn_id]` when they are
  replaced
- if commit fails before metadata publication, the old freelist remains
  authoritative
- metadata publication is the only point where promotions and allocations become
  durable

Restart rules for M6 single-process mode:

- `open()` reloads only the freelist referenced by selected metadata
- persisted pending groups remain grouped by `freeing_txn_id`
- a later writer may promote all eligible groups because no in-process readers
  survive restart
- ignored tail pages outside selected metadata stay ignored and do not join
  `reusable` or `pending_free`

## 8. Tests

Required test areas:

| Area | Scenario | Expected Result |
| --- | --- | --- |
| reusable model | decode freelist with reusable ranges and pending groups | exact reusable and pending counts are rebuilt |
| pending promotion | no readers active, writer starts | eligible committed pending groups move into transaction-local reusable |
| pending promotion | reader with snapshot lower than `X` active | `pending_free[X]` stays blocked |
| pending promotion | reader with snapshot equal to `X` active | `pending_free[X]` is promotable |
| same-tx no reuse | writer deletes pages then allocates again | pages freed in the same transaction are not reused |
| allocation order | reusable range available for request | allocator reuses pages before tail growth |
| allocation order | only fragmented reusable pages remain | allocator reuses fragmented pages before tail growth |
| allocation order | reusable exhausted | allocator grows the tail and advances logical `page_count` |
| overflow reuse | replace large value after older reader closes | later writer reuses eligible overflow pages |
| tree reuse | delete/update tree pages across writes | later writer reuses eligible branch/leaf pages |
| freelist self-accounting | commit writes replacement freelist image | old freelist pages enter `pending_free[new_txn_id]`; new freelist pages are not free |
| duplicate allocation guard | repeated allocations from one reusable pool | no page ID is returned twice |
| rollback boundary | delete then rollback | `freed_in_tx` is dropped and no freelist change is published |
| restart promotion | close and reopen single-process DB with persisted pending groups | next writer can promote eligible groups without scanning unreachable pages |
| crash boundary | crash after freelist write before metadata | old freelist remains authoritative |
| crash boundary | crash after metadata selects promoted/reused pages | new freelist image is authoritative |

Test naming should stay behavior-oriented, for example:

- `pending_group_promotes_when_oldest_reader_is_equal_to_freeing_txn`
- `allocator_does_not_reuse_pages_freed_by_same_write_transaction`
- `writer_reuses_reusable_pages_before_growing_logical_tail`

## 9. M7 Handoff

M6 should leave explicit extension points rather than baking the single-process
policy into allocation and validation code.

Required handoff seams:

- expose `oldest_reader_txn_id()` through a small internal provider that M7 can
  replace with cross-process reader visibility
- keep reusable/pending validation helpers independent from the single-process
  registry implementation
- preserve grouped `pending_free[freeing_txn_id]` semantics so M7 verify and
  compaction can classify blocked versus reusable pages accurately
- keep allocation policy generic across tree, overflow, and freelist pages so
  M7 compaction can reuse the same accounting model
- expose read-only walkers or validators for freelist images that M7 verify can
  call without reimplementing decoding logic

What M7 should inherit unchanged:

- pending groups are keyed by freeing transaction ID
- promotion is based on a reader visibility cutoff, not reachability scans
- same-transaction freed pages are never reusable
- durable reuse state changes happen only through the normal metadata-switched
  commit path

What M7 may replace or extend:

- the source of `oldest_reader_txn_id`
- public verify tooling over the existing freelist validators
- compaction or file-shrinking flows that consume, but do not redefine, M6 free
  space accounting

## 10. Acceptance

This feature is accepted when:

- the implementation persists both `reusable` and grouped
  `pending_free[freeing_txn_id]` state through the normal freelist commit path
- pending groups promote to reusable only when the oldest-reader cutoff proves
  no active reader has `snapshot_txn_id < freeing_txn_id`
- a reader at snapshot `X` does not block promotion of `pending_free[X]`, while
  an older reader does block it
- pages freed by the active writer are never reused by that same transaction
- the allocator reuses eligible reusable pages before growing the logical file
  tail
- blocked pending groups, fixed pages, and the active writer's freed pages are
  never returned by the allocator
- old tree, overflow, and freelist pages all flow through the same
  `pending_free -> reusable -> allocate` lifecycle
- rollback and pre-publication commit failure do not publish promotions,
  allocations, or newly freed pages
- restart in single-process mode can promote eligible persisted pending groups
  without inferring free pages from ignored tail space or unreachable disk pages
- tests cover reusable/pending models, oldest-reader cutoff behavior,
  same-transaction no-reuse semantics, allocation order, crash boundaries, and
  restart behavior
- the resulting implementation leaves clear M7 seams for cross-process reader
  tracking, verify tooling, and compaction without changing M6 on-disk
  semantics
