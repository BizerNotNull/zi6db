# Dirty Page Ownership And Mutation Isolation

This feature defines the concrete M3 implementation plan for transaction-local
dirty page ownership, copy-on-write mutation isolation, and the allocator/cache
rules that keep uncommitted state invisible. It refines the M3 transaction
plan in `docs/design/m3/m3-implementation-plan.md` and reuses the M2 mutation
builder seam defined in
`docs/design/m2/features/copy-on-write-mutation-and-splits.md`.

The goal is to make M3 transaction semantics correct in-process without adding
page lifecycle shortcuts that M4 crash safety or M6 overflow-page work would
need to replace later.

## 1. Scope

In scope:

- transaction-local tracking for `dirty_pages`, `copied_pages`, and `new_pages`
- mutation ownership rules for cloned, split, and newly allocated pages
- `page_generation` assignment for pending commits
- isolation between transaction-private dirty state and shared committed cache
- allocation rules for tail growth, private reuse, and committed `page_count`
- rollback cleanup for all private write state
- tests for visibility, ownership, allocation gaps, and cache isolation
- explicit handoff seams for M4 durability/recovery and M6 overflow pages

Out of scope:

- changing the M0 on-disk format
- cross-process locking or writer coordination
- new cursor or range-scan APIs
- freelist compaction or file shrinkage
- overflow-page encoding itself

## 2. Core Intent

M3 must enforce one ownership rule above all others:

- committed pages are shared and immutable
- dirty pages are private to exactly one active `WriteTx`
- read transactions can observe only committed pages reachable from their pinned
  metadata snapshot

This means the implementation must separate three concepts that M2 could treat
more loosely:

- visible committed page identity
- transaction-private mutated page identity
- allocator bookkeeping for page ids that exist only inside the writer

The mutation path should therefore be:

1. start from the writer base metadata snapshot
2. clone or allocate private pages into transaction state
3. mutate only private pages
4. finalize page headers, `page_generation`, and checksums before writeout
5. publish the private graph only by writing the inactive metadata slot

## 3. Transaction-Local Structures

Each active `WriteTx` should own the following structures:

### `dirty_pages`

`dirty_pages` is the authoritative private page table for this writer.

Required shape:

- keyed by destination page id
- stores finalized-or-finalizable page buffers for branch, leaf, and later
  overflow/freelist pages
- never shared with read transactions
- never inserted into the shared committed-page cache before commit success

Required invariants:

- every page id in `dirty_pages` belongs to this transaction only
- a dirty page buffer is the only mutable copy of that destination page id
- mutations after first staging must update the existing buffer, not fork a
  second buffer for the same page id
- writeout order may be tracked separately, but ownership comes from
  `dirty_pages`

### `copied_pages`

`copied_pages` records the copy-on-write mapping from old visible page id to
new private page id.

Required shape:

- keyed by source committed page id
- value is the destination page id of the private clone
- populated only for pages copied from the base visible tree

Required invariants:

- cloning the same visible page twice in one transaction must return the same
  private clone
- tree mutation helpers must consult `copied_pages` before allocating a second
  clone
- `copied_pages` is a routing/identity table, not the page storage itself; the
  actual mutable buffer remains in `dirty_pages`

### `new_pages`

`new_pages` tracks page ids allocated fresh during this transaction, whether
they came from tail allocation or from a reusable page proven safe to claim.

Required shape:

- set or map keyed by newly owned page id
- includes split siblings, new roots, new freelist pages, and later overflow
  pages
- includes private allocations not derived from cloning a committed page

Required invariants:

- every page id in `new_pages` is reserved for this transaction until commit or
  rollback
- `new_pages` must be sufficient to release all private allocation ownership on
  rollback
- a page may appear in both `new_pages` and `dirty_pages`, but their meanings
  differ: allocation ownership versus staged mutable bytes

### Optional `abandoned_new_pages`

If the allocator supports transaction-local recycling before commit, keep a
private `abandoned_new_pages` pool. If not, the implementation may fold this
state into `new_pages` plus a per-page status flag.

Required invariant:

- abandoned pages may be reused only by the same active writer and must never
  become committed freelist state by accident

## 4. Ownership Rules During Mutation

All B+Tree mutating helpers must require `WriteTx` access and obey these rules:

1. never mutate a committed visible page in place
2. when a visible page must change, consult `copied_pages`
3. if no clone exists, allocate a private page id and stage a clone
4. store the mutable page buffer in `dirty_pages`
5. record the source-to-clone mapping in `copied_pages`
6. if a split creates a brand-new sibling, allocate it directly into
   `new_pages` and `dirty_pages`
7. if a new root is required, allocate it directly as a private page

Additional rules:

- branch and leaf mutations must update only the private clone chain rooted at
  `WriteTx.new_root_page_id`
- a root pointer change updates transaction state only; it does not affect
  `DB.current_metadata`
- deleted committed pages are added to `pending_free_delta` for the future
  committed `txn_id`; they do not become immediately reusable
- the inactive metadata page is staged outside normal tree dirty-page state and
  must not appear in read traversal or shared cache lookup

## 5. `page_generation` Plan

M3 must assign the commit generation to dirty normal pages before computing
their checksums.

Implementation steps:

1. enter commit with `base_generation`
2. compute `new_generation = base_generation + 1`
3. finalize every dirty branch, leaf, and later overflow page with
   `page_generation = new_generation`
4. compute checksums after the generation field is set
5. write non-metadata pages
6. build metadata carrying the same `new_generation`

Required invariants:

- no committed normal page written by a successful transaction carries the old
  generation
- page checksum coverage must include the finalized generation field
- generation is a commit-time property, not a mutation-planning placeholder
- staged pages must not be visible through committed read paths even if their
  `page_generation` is already finalized

Validation requirements:

- tests should inspect staged dirty-page headers before writeout where hooks are
  available
- successful commit tests must verify metadata `generation` and written normal
  page `page_generation` advance together

## 6. Cache Isolation

M3 must draw a strict line between the shared committed cache and
transaction-private dirty state.

Preferred design:

- keep `dirty_pages` entirely outside the shared page cache
- let read transactions consult only committed cache entries plus storage reads
- let the active writer consult committed pages for clone sources and its own
  `dirty_pages` for private state

Acceptable alternative:

- if one unified cache structure is used, dirty entries must be keyed by both
  page id and transaction owner so read paths can never resolve them by page id
  alone

Rules that must hold either way:

- a read transaction must never receive a pointer or slice backed by a dirty
  page buffer
- a rolled-back or failed writer must remove all dirty cache entries before the
  writer gate is released
- commit success may publish written pages into the committed cache only after
  the metadata boundary succeeds and in-memory metadata switches
- old committed cache entries remain valid for readers pinned to older metadata
  until those readers close or cache eviction removes the entries naturally

Practical read-path rule:

- `get()` returns caller-owned memory copied from committed page contents; M3
  must not expose borrowed cache slices because page/cache lifetime rules are
  not yet public API

## 7. Allocation And Committed `page_count`

The allocator must distinguish:

- visible committed pages from the writer base snapshot
- private pages allocated by the active transaction
- reusable committed pages proven safe by freelist and reader-registry rules

### Allocation order

The safest default for M3 is:

1. use reusable pages that are already eligible under the pinned freelist and
   reader cutoff rules, if that path is implemented cleanly
2. otherwise allocate from the tail

M3 must not speculate about reuse by scanning for unreachable pages.

### Allocation gaps

The most important `page_count` rule is:

- a page id allocated privately but not ultimately published must not create a
  committed gap that looks allocated yet has no owned meaning

This means an abandoned page id must follow one of two legal outcomes:

- it is reused later by the same transaction for another dirty page and still
  falls below the committed `new_page_count`
- it remains uncommitted and `new_page_count` stays below that page id

Illegal outcome:

- `new_page_count` advances past an abandoned page id that is neither written as
  committed content nor represented as reusable/pending-free freelist state

### Allocation bookkeeping requirements

Implementation should track at least:

- `allocated_page_count_base`
- `highest_private_page_id`
- whether each privately allocated page ended as written, recycled privately, or
  abandoned outside the commit result

Commit-time check:

- before metadata is built, assert that `new_page_count` is the exclusive upper
  bound of committed page ids intended to exist after the transaction

Rollback-time check:

- private allocations vanish from transaction ownership with no persistent
  freelist mutation and no file repair

## 8. Rollback Cleanup

Rollback must discard ownership, not repair bytes on disk.

Required cleanup sequence:

1. block further writes through the transaction
2. drop all `dirty_pages` buffers
3. clear `copied_pages`
4. clear `new_pages`
5. clear any `abandoned_new_pages` pool
6. clear staged root/freelist/page-count updates
7. clear `pending_free_delta`
8. remove any private cache state
9. release the writer gate

Required outcomes:

- `DB.current_metadata` is unchanged
- existing readers continue using their pinned committed pages
- no rolled-back allocation becomes committed or reusable
- no later writer can observe stale dirty buffers

## 9. Implementation Order

Implement this feature in the following order:

1. define `WriteTx` fields for `dirty_pages`, `copied_pages`, `new_pages`, and
   allocation bookkeeping
2. refactor M2 mutation helpers so every mutating page operation requires
   `WriteTx`
3. add clone helpers that first consult `copied_pages`
4. add new-page allocation helpers that register ownership in `new_pages`
5. split shared cache access from transaction-private dirty lookup
6. add staged-page finalization for `page_generation` and checksum ordering
7. add rollback cleanup for all private ownership state
8. add commit assertions for `new_page_count` and cache publication ordering
9. add tests for ownership, isolation, and allocation-gap prevention

## 10. Test Matrix

Required tests:

| Area | Scenario | Expected result |
| --- | --- | --- |
| `copied_pages` | mutate the same committed leaf twice in one write transaction | second mutation reuses the first clone |
| `dirty_pages` | mutate cloned leaf and parent branch | both pages exist only in transaction-private dirty state |
| `new_pages` | split leaf then split root | sibling and root page ids are tracked as newly owned pages |
| dirty visibility | reader runs while writer updates key before commit | reader sees only committed value |
| cache isolation | shared cache lookup during active write | no dirty buffer is returned to read path |
| cache cleanup | rollback writer with staged dirty pages | dirty cache/private state is fully removed |
| cache cleanup | injected commit failure before metadata switch | dirty cache/private state is removed and in-memory metadata is unchanged |
| `page_generation` | finalize dirty normal pages during commit | every dirty normal page carries pending `new_generation` before checksum |
| allocation reuse | allocator abandons a private page then reuses it in same transaction | committed `page_count` remains valid and gap-free |
| allocation gap | allocator abandons a private tail page and does not reuse it | committed `new_page_count` stays below abandoned page id |
| rollback | rollback after clone, split, and delete staging | metadata, visible root, and freelist state remain unchanged |
| pending free | delete committed pages while older reader is active | deleted pages stay in pending-free delta, not reusable |
| commit publish | successful commit | committed cache/in-memory metadata switch only after metadata boundary |
| ownership boundary | helper attempts mutation without `WriteTx` | compile-time or runtime rejection |

Recommended test groups:

- `dirty_page_ownership`
- `mutation_isolation`
- `allocation_gap_rules`
- `cache_visibility`
- `page_generation_finalization`

## 11. M4 Handoff

This feature must leave M4 with durable seams, not redesign work.

M4 should be able to add:

- fault injection around non-metadata page writes, metadata writes, and syncs
- crash/reopen verification for ambiguous metadata write outcomes
- recovery-time metadata selection using existing A/B slot behavior
- stricter validation that staged pages were finalized before writeout

M3 therefore must already guarantee:

- dirty-page ownership is transaction-local and explicit
- metadata publication is the only visibility switch
- normal dirty pages carry finalized `page_generation` before checksum/writeout
- in-memory metadata does not switch on pre-metadata failure
- private abandoned pages are not silently published as committed allocation gaps

## 12. M6 Handoff

M6 adds overflow pages, but it must inherit the same ownership model.

M3 should therefore treat later overflow pages as another private page class
rather than a special exception. The future M6 implementation must be able to:

- allocate overflow pages into `new_pages`
- stage overflow page buffers in `dirty_pages`
- link overflow ownership to the same rollback and commit cleanup paths
- assign overflow `page_generation` at commit finalization time
- keep overflow dirty buffers out of the shared committed cache before commit

If M3 implements ownership generically for page classes instead of only branch
and leaf pages, M6 can extend the same machinery without changing semantics.

## 13. Acceptance

This feature is accepted when:

- `dirty_pages`, `copied_pages`, and `new_pages` each have explicit ownership
  roles and no overlapping ambiguity
- every mutating tree helper requires `WriteTx` ownership and never edits a
  committed visible page in place
- repeated mutation of the same committed page reuses one clone through
  `copied_pages`
- dirty pages remain invisible to all read transactions until metadata publish
- shared cache access cannot leak dirty buffers into committed read paths
- rollback and failed commit cleanup remove all private dirty/cache state before
  another writer can proceed
- committed normal pages receive the pending commit `page_generation` before
  checksum calculation
- private allocations cannot create committed `page_count` gaps
- deleted committed pages flow into pending-free state instead of direct reuse
- the implementation exposes clean seams for M4 durability/recovery and M6
  overflow pages without replacing the ownership model
- tests cover ownership, isolation, generation finalization, allocation gaps,
  rollback cleanup, and handoff boundaries
