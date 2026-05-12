# Rollback Cleanup And Freelist Reader Safety

This feature plan narrows the M3 transaction work for rollback cleanup,
`pending_free_delta`, pending-to-reusable promotion, and reader-safety cutoffs.
It is derived from
`docs/design/m3/m3-implementation-plan.md`,
`docs/design/m6/m6-implementation-plan.md`,
and the repository rules in `AGENTS.md`.

The goal is to make M3 transaction rollback and page-reuse boundaries concrete
without pulling M6 overflow or full space-reuse scope forward. M3 must leave a
correct, testable in-process model where:

- rollback discards only private writer state
- freed pages stay grouped in `pending_free_delta` until commit
- pending pages are promoted only when the reader cutoff proves reuse is safe
- the allocator never reuses a page that an older reader may still reach
- M6 can extend the same rules to overflow pages and a complete freelist

## 1. Scope

In scope:

- writer rollback cleanup steps and state transitions
- transaction-local `pending_free_delta` structure and ownership rules
- promotion rules from persisted `pending_free` to reusable pages
- reader registry cutoff logic used by writer allocation and commit
- tests that prove rollback and reader-safety behavior
- explicit M6 handoff seams for full freelist reuse and overflow integration

Out of scope:

- cross-process reader visibility or lock tables
- background reclamation
- file truncation, compaction, or vacuum
- overflow chain freeing and reuse logic
- full crash-recovery verification beyond M3 hook-level boundaries

## 2. Core Intent

M3 should treat page reuse as a transaction-boundary safety problem, not a
storage scavenging problem.

Three rules drive the implementation:

1. A writer may make pages unreachable in its staged tree, but those pages do
   not become reusable inside the same transaction.
2. A committed delete publishes freed pages only as
   `pending_free[new_txn_id]`, never directly as reusable pages.
3. A page becomes reusable only in a later transaction that proves no active
   reader has `snapshot_txn_id < freeing_txn_id`.

This keeps M3 compatible with the M0 metadata-switch model and gives M6 a safe
base for true space reuse.

## 3. `pending_free_delta`

`pending_free_delta` is the write transaction's private record of pages that
became unreachable from the staged post-mutation tree.

Recommended shape:

- store page IDs grouped under the future commit transaction id
- represent the working value as a transaction-local list or range set
- treat the group as logically destined for `pending_free[new_txn_id]`
- keep it separate from the persisted freelist snapshot until commit finalizes

Sources of entries:

- cloned branch pages replaced by newer copies
- cloned leaf pages replaced by newer copies
- old root pages replaced by a new root
- old freelist pages replaced by a new freelist image, if M3 already rewrites
  the freelist page

Rules:

- add only pages that were visible in the writer's base snapshot and are no
  longer reachable from the staged root
- do not add pages newly allocated by the same transaction and later abandoned
- do not mix committed pending groups from the base freelist into
  `pending_free_delta`
- do not persist `pending_free_delta` until the commit path assigns
  `new_txn_id`
- do not allow allocation from `pending_free_delta` during the same
  transaction

Implementation note:

`pending_free_delta` should be attached to the mutation result or writer state,
not derived by rescanning dirty pages during commit. The mutation path already
knows which visible pages it replaced, so it should record that set once.

## 4. Rollback Steps

Rollback must be a pure in-memory cleanup path. It must not repair disk,
re-encode committed pages, or try to infer reusable space.

`rollback(WriteTx)` should run these steps in order:

1. Validate that the transaction is still in `active_write`.
2. Flip the transaction into a closing state or otherwise block further reads
   and writes through that handle.
3. Drop dirty branch and leaf page buffers.
4. Drop any staged freelist page buffers.
5. Drop `copied_pages` mappings from visible page ID to private clone ID.
6. Drop `new_pages` and any transaction-local allocation bookkeeping.
7. Drop `pending_free_delta`.
8. Drop staged root, freelist, and page-count replacements.
9. Clear any transaction-local reusable-page reservations derived from the base
   freelist snapshot.
10. Release the writer gate.
11. Mark the transaction `closed_rollback`.

Required invariants after rollback:

- `DB.current_metadata` is unchanged
- the reader registry is unchanged
- no page write, metadata write, sync, truncate, or recovery scan occurs
- no page from `pending_free_delta` becomes visible in reusable state
- later readers see the same metadata snapshot they would have seen before the
  rolled-back writer began

Lifecycle rule:

Rollback is valid only before commit enters `committing`. Once commit starts,
the transaction must be treated as terminal; rollback cannot be used to make an
ambiguous commit "more rolled back."

## 5. Promotion Rules

M3 needs one conservative promotion rule and should apply it consistently:

- a pending group with `freeing_txn_id = X` may move to reusable only when no
  active reader has `snapshot_txn_id < X`

Equivalent cutoff form:

- if `oldest_reader_txn_id` exists, group `X` is promotable when
  `oldest_reader_txn_id >= X`
- if no readers are active, all committed pending groups from the writer's base
  freelist snapshot are promotable

Promotion timing:

- promotion may run at `beginWrite()` when building the writer's allocator view
- promotion may run again during commit finalization before the new freelist
  image is serialized
- promotion becomes durable only when the new freelist page is committed through
  metadata publication

Rules:

- never promote the current transaction's own `pending_free_delta`
- preserve `freeing_txn_id` grouping for every not-yet-eligible base pending
  group
- never infer free pages by walking unreachable disk pages
- never mutate the base snapshot freelist in place

Practical M3 recommendation:

Build a transaction-local allocator view from the writer's pinned freelist
snapshot:

- `base_reusable`
- `base_pending_promotable`
- `base_pending_blocked`

Then:

- merge `base_pending_promotable` into the transaction-local reusable set
- keep `base_pending_blocked` grouped and unchanged for the new freelist image
- append `pending_free_delta` under `new_txn_id` during commit

## 6. Reader Safety Cutoff

The reader registry exists to answer one question safely:

- what is the oldest `snapshot_txn_id` among currently active readers?

Recommended in-memory model:

- maintain an exact active reader count
- maintain per-reader identity so duplicate `snapshot_txn_id` values can be
  removed one reader at a time
- maintain an `oldest_reader_txn_id()` helper or equivalent min-tracking path

Registration rule:

- metadata snapshot capture and reader registration must happen in one critical
  section

Removal rule:

- read transaction close or rollback must remove exactly one registry entry

Cutoff rule for writers:

- allocation from committed reusable pages is allowed only from the writer's
  transaction-local reusable view after applying the reader cutoff
- pages from blocked pending groups remain unavailable even if they are
  unreachable from current metadata

Why equality is safe:

- if a pending group has `freeing_txn_id = X`, a reader with snapshot `X`
  started after transaction `X` committed and therefore cannot still reach the
  pre-delete page versions from before `X`
- only readers with snapshot ids lower than `X` can still require those old page
  versions

M3 safety boundary:

- the reader cutoff is authoritative for in-process safety
- after restart, there are no surviving in-process readers, but promotion still
  becomes durable only through a later committed freelist rewrite

## 7. Allocation And Cleanup Rules

Even if M3 initially prefers tail allocation, the allocator should already obey
future freelist safety semantics.

Allocation rules:

- never allocate pages `0`, `1`, or `2`
- never reuse a page from the writer's own `pending_free_delta`
- never reuse from a base pending group blocked by the reader cutoff
- remove reused page IDs from the transaction-local reusable set immediately
- keep newly allocated page IDs tracked in `new_pages` until commit or rollback

Cleanup rules for failed or rolled-back writers:

- private reusable reservations return only to the in-memory future writer view
- no rollback or pre-publication commit failure may modify persisted freelist
  state
- unreachable tail pages created by a failed writer must remain ignored unless a
  later committed metadata snapshot references them

## 8. Tests

Required executable test areas:

| Area | Scenario | Expected Result |
| --- | --- | --- |
| rollback cleanup | insert then rollback | key is absent for a new reader; no metadata switch |
| rollback cleanup | update then rollback | old value remains visible |
| rollback cleanup | delete then rollback | key remains visible |
| rollback cleanup | split-producing write then rollback | root and lookups match pre-write state |
| rollback cleanup | rollback after private page allocations | writer releases state without disk writes |
| rollback lifecycle | reuse rolled-back writer | `TransactionClosed` |
| pending delta | delete visible pages in one write tx | pages appear only in `pending_free_delta` before commit |
| pending delta | allocate after delete in same tx | deleted pages are not reused |
| promotion | no readers active, writer starts | eligible base pending groups move into transaction-local reusable |
| promotion | reader with snapshot lower than `X` active | group `X` stays blocked |
| promotion | reader with snapshot equal to `X` active | group `X` is promotable |
| reader registry | two readers share same snapshot id | closing one removes only one registry entry |
| reader cutoff | old reader active while later writer deletes page | page remains pending, not reusable |
| reader cutoff | old reader closes before next writer | next writer may reuse previously pending pages |
| allocator safety | writer reuses committed reusable page | page is removed from reusable before allocation |
| commit integration | commit appends `pending_free_delta` under `new_txn_id` | new freelist image preserves grouping |
| failure boundary | injected commit failure before metadata write | base metadata and base freelist remain authoritative |

Test design notes:

- use behavior-oriented names such as
  `rollback_drops_pending_free_delta_without_publishing_it`
- assert both logical state and lifecycle state
- where real disk-side failure injection is difficult, test through storage
  hooks already planned for M3 commit ordering

## 9. M6 Handoff

M6 will expand these M3 rules to full space reuse and overflow pages. M3 should
therefore leave explicit seams instead of a one-off implementation.

Required handoff seams:

- `pending_free_delta` is already a first-class transaction field
- writer commit accepts a transaction-local promoted/blocking view of the base
  freelist rather than mutating global state directly
- the reader registry exposes `oldest_reader_txn_id()` through a small internal
  interface
- allocator logic does not special-case tree pages in a way that would block
  future overflow-page reuse
- rollback cleanup can drop private overflow allocations later without changing
  its shape
- commit can append additional freed page classes, including overflow and old
  freelist pages, into the same `pending_free[new_txn_id]` model

What M6 should inherit unchanged:

- same-transaction freed pages are never reusable
- pending groups are keyed by freeing transaction id
- promotion is based on reader cutoff, not reachability scans
- durable freelist state changes happen only through normal metadata-switched
  commit

What M6 will add on top:

- overflow chain enumeration and freeing
- allocation from reusable before tail growth as a production policy
- richer freelist serialization, likely with ranges and multi-page freelist
  images
- restart-time promotion of eligible pending groups in single-process mode

## 10. Acceptance

This feature is accepted when:

- `rollback(WriteTx)` drops all private dirty, copied, allocated, and
  `pending_free_delta` state without writing to disk.
- Rollback leaves `DB.current_metadata` unchanged and releases the writer gate.
- Pages deleted or replaced by a write transaction accumulate in
  `pending_free_delta` and are not reusable inside the same transaction.
- Commit appends `pending_free_delta` as `pending_free[new_txn_id]` rather than
  flattening it into reusable pages.
- Pending groups promote to reusable only when the reader cutoff proves
  `snapshot_txn_id < freeing_txn_id` is false for all active readers.
- A reader at snapshot `X` does not block promotion of `pending_free[X]`, while
  a reader older than `X` does block it.
- The allocator never returns page IDs from blocked pending groups or the active
  writer's own freed-page delta.
- Reader registration and metadata snapshot capture are atomic.
- Reader deregistration removes exactly one reader record, even when multiple
  readers share the same snapshot transaction id.
- Tests cover rollback correctness, lifecycle closure, pending-group promotion,
  reader cutoff behavior, and commit integration boundaries.
- The resulting implementation can be extended in M6 to overflow pages and full
  freelist reuse without changing the M3 transaction or reader-safety model.
