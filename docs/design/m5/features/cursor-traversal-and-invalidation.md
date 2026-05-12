# Cursor Traversal And Invalidation

This document defines the implementation plan for the M5 feature `cursor traversal and invalidation`. It refines [m5-implementation-plan.md](/D:/代码D/zi6db/docs/design/m5/m5-implementation-plan.md), follows the transaction lifecycle rules in [m3-implementation-plan.md](/D:/代码D/zi6db/docs/design/m3/m3-implementation-plan.md), and stays within the repository guidance in [AGENTS.md](/D:/代码D/zi6db/AGENTS.md).

## 1. Scope

This feature covers the concrete M5 behavior for:

- `first()`
- `last()`
- `seek(target)`
- `next()`
- `prev()`
- cursor path state from root to current leaf
- no-item positioning semantics
- write-transaction cursor invalidation after same-bucket mutation
- tests and acceptance criteria

This feature does not define:

- bucket catalog encoding
- range or prefix wrapper APIs
- overflow-page traversal rules
- mutation-while-iterating recovery behavior
- new on-disk page formats

## 2. Required Invariants

The implementation must preserve these invariants:

- a cursor lifetime is bound to its owning transaction
- a cursor is bucket-local and must never be reused for another bucket
- read-transaction cursors observe only the snapshot pinned at transaction begin
- write-transaction cursors observe the writer's staged bucket root at creation time
- `key()` and `value()` return `null` only for normal no-item state
- closed or invalidated cursors fail with `TransactionClosed` or `InvalidState`
- write-transaction bucket mutations invalidate existing cursors for that bucket
- `dropBucket()` invalidates existing cursors for that bucket immediately
- cursor movement must remain correct across multi-level and multi-leaf trees
- traversal correctness must not depend on leaf sibling links if parent-path traversal can produce the same result

## 3. Cursor State Model

### 3.1 Required Fields

Each cursor should store enough state to resume traversal without restarting from
the root on every `next()` or `prev()` call:

```zig
const Cursor = struct {
    tx: *Tx,
    bucket_id: BucketIdentity,
    bucket_root_page_id: u64,
    bucket_mutation_revision: u64,
    state: CursorState,
    path: std.ArrayListUnmanaged(PathFrame),
    leaf_slot_index: u16,
};

const PathFrame = struct {
    page_id: u64,
    child_index: u16,
};

const CursorState = enum {
    no_item,
    positioned,
    invalidated,
};
```

The exact field names may change, but the implementation must keep these
logical facts:

- the owning transaction
- the bucket snapshot or staged root this cursor was created against
- the bucket-local mutation revision visible at creation time
- a root-to-leaf traversal path
- the current slot inside the current leaf
- an explicit distinction between positioned, no-item, and invalidated

### 3.2 Path Semantics

The path must describe the currently positioned leaf in a way that supports:

- stepping to the next leaf by walking up to the first ancestor with a larger
  child branch and descending again to the leftmost leaf under that branch
- stepping to the previous leaf by walking up to the first ancestor with a
  smaller child branch and descending again to the rightmost leaf under that
  branch
- rebuilding the leaf position after `first()`, `last()`, or `seek()`

Implementation rules:

- when `state == .positioned`, the path is non-empty and ends at a leaf page
- when `state == .no_item`, the path must be cleared
- when `state == .invalidated`, later public operations fail before consulting
  the path
- `leaf_slot_index` is valid only when `state == .positioned`
- path frames must refer to the committed or staged tree visible to the cursor,
  never to writer-private pages from another transaction

## 4. No-Item Semantics

M5 needs one explicit off-tree state instead of overloading lifecycle errors.

Normal no-item cases:

- `first()` on an empty bucket
- `last()` on an empty bucket
- `seek(target)` when the bucket is empty
- `seek(target)` when `target` is greater than every key in the bucket
- `next()` from the last item
- `prev()` from the first item
- `next()` or `prev()` while already in no-item state

Observable rules:

- `key()` returns `null`
- `value()` returns `null`
- repeated `next()` stays in no-item state
- repeated `prev()` stays in no-item state
- no-item is not an error and must remain distinguishable from invalidation

M5 must not expose hidden recovery behavior from no-item state. In particular:

- `seek(after_last)` then `prev()` remains no-item
- `first()` on an empty bucket then `next()` remains no-item
- `last()` on an empty bucket then `prev()` remains no-item

If a later milestone wants "seek past end, then step backward" behavior, it
should add a separate helper such as `seekLe()` rather than change M5 cursor
state semantics.

## 5. Movement Semantics

### 5.1 `first()`

`first()` must:

1. validate transaction and cursor lifecycle
2. validate the bucket mutation revision
3. descend from the visible bucket root through child index `0` until a leaf is
   reached
4. position on slot `0` in that leaf if the leaf contains entries
5. otherwise enter no-item state

Expected result:

- smallest key in the bucket when non-empty
- no-item state when empty

### 5.2 `last()`

`last()` must:

1. validate transaction and cursor lifecycle
2. validate the bucket mutation revision
3. descend from the visible bucket root through the last child at each branch
4. position on the last occupied slot in the leaf if the leaf contains entries
5. otherwise enter no-item state

Expected result:

- largest key in the bucket when non-empty
- no-item state when empty

### 5.3 `seek(target)`

`seek(target)` must position at the first key greater than or equal to
`target`.

Algorithm shape:

1. validate lifecycle and revision
2. descend from the root while choosing the first child that can contain a key
   `>= target`
3. at the leaf, binary-search for the first slot whose key is `>= target`
4. if found, position there
5. if not found in that leaf, advance to the next leaf using the parent path
6. if no later leaf exists, enter no-item state

Expected result:

- exact-match key when present
- next larger key when `target` falls in a gap
- first key when `target` is smaller than every key
- no-item state when `target` is larger than every key

### 5.4 `next()`

`next()` must:

- fail on invalidated or closed cursor state
- remain in no-item state when already no-item
- advance within the current leaf when another slot exists
- otherwise walk the parent path to the next reachable leaf and position at its
  first slot
- enter no-item state when no later leaf exists

### 5.5 `prev()`

`prev()` must:

- fail on invalidated or closed cursor state
- remain in no-item state when already no-item
- move backward within the current leaf when a lower slot exists
- otherwise walk the parent path to the previous reachable leaf and position at
  its last slot
- enter no-item state when no earlier leaf exists

## 6. Write-Transaction Invalidation

### 6.1 Conservative Rule

M5 should implement the conservative invalidation model from the milestone plan:

- read-transaction cursors stay valid until the read transaction closes
- write-transaction cursors for a bucket are invalidated by any `put()` in that
  bucket
- write-transaction cursors for a bucket are invalidated by any `delete()` in
  that bucket
- write-transaction cursors for a bucket are invalidated by `dropBucket()` for
  that bucket
- newly created cursors after the mutation must observe the latest staged root

This avoids promising best-effort continuation across copy-on-write page
relocation.

### 6.2 Revision Tracking

Each bucket visible to a write transaction should expose a monotonic
`mutation_revision`.

Rules:

- creating a cursor captures the bucket's current revision
- `put()` increments the bucket revision even if it updates an existing key
- `delete()` increments the bucket revision only when it actually removes a key
- `dropBucket()` increments or tombstones the bucket state so existing cursors
  fail immediately
- every public cursor movement or accessor checks the captured revision against
  the bucket's current revision before using the stored path

Error behavior:

- transaction closed remains `TransactionClosed`
- bucket dropped or same-bucket write mutation becomes `InvalidState`
- no-item remains a normal state and does not use errors

### 6.3 Scope Of Invalidation

Invalidation must be bucket-local inside a write transaction.

Required behavior:

- mutating bucket `a` invalidates cursors created from bucket `a`
- mutating bucket `a` does not invalidate cursors created from bucket `b`
- catalog mutations invalidate catalog traversal used by `listBuckets()`
- a writer may create a new cursor after mutation and that new cursor must work
  against the new staged bucket root

## 7. Implementation Steps

Implement this feature in the following order:

1. Add cursor structs, path frames, and explicit cursor state enums.
2. Add lifecycle guards for closed transactions, dropped buckets, and invalid
   cursor state.
3. Add bucket-local write-transaction mutation revisions.
4. Implement root-to-leftmost descent for `first()`.
5. Implement root-to-rightmost descent for `last()`.
6. Implement leaf lower-bound search and parent-path capture for `seek()`.
7. Implement cross-leaf forward stepping for `next()` using the parent path.
8. Implement cross-leaf backward stepping for `prev()` using the parent path.
9. Clear path state consistently when transitioning to no-item.
10. Validate that `key()` and `value()` distinguish no-item from invalidated
    state.
11. Add multi-leaf tests, mutation invalidation tests, and transaction lifetime
    tests.
12. Reuse the same internal path helpers later for M5 range and prefix cursors
    without changing the public no-item semantics.

## 8. Test Plan

| Area | Scenario | Expected result |
| --- | --- | --- |
| Empty bucket | `first()` on empty bucket | no-item, `key()` and `value()` return `null` |
| Empty bucket | `last()` on empty bucket | no-item, `key()` and `value()` return `null` |
| Empty bucket | `seek("x")` on empty bucket | no-item |
| Forward iteration | keys `a`, `b`, `c` | `first()` then repeated `next()` yields `a`, `b`, `c`, then no-item |
| Backward iteration | keys `a`, `b`, `c` | `last()` then repeated `prev()` yields `c`, `b`, `a`, then no-item |
| Seek exact | `seek("b")` in `a`, `b`, `c` | positioned at `b` |
| Seek gap | `seek("bb")` in `a`, `b`, `c` | positioned at `c` |
| Seek before first | `seek("0")` in `a`, `b` | positioned at `a` |
| Seek after last | `seek("z")` in `a`, `b` | no-item |
| No-item stability | `seek("z")`, then `prev()` | remains no-item |
| No-item stability | `first()` on empty, then `next()` | remains no-item |
| No-item stability | `last()` on empty, then `prev()` | remains no-item |
| Multi-leaf next | iterate across a leaf split boundary | every key appears once in order |
| Multi-leaf prev | iterate backward across a leaf split boundary | every key appears once in reverse order |
| Read snapshot | reader cursor open, writer commits later change | reader cursor keeps old snapshot |
| Writer read-your-writes | writer mutates bucket, creates new cursor | new cursor sees staged write |
| Write invalidation put | writer cursor exists, then `put()` same bucket | next cursor operation returns `InvalidState` |
| Write invalidation delete existing | writer cursor exists, then successful `delete()` same bucket | next cursor operation returns `InvalidState` |
| Write invalidation delete missing | writer cursor exists, `delete()` missing key | cursor remains valid because no mutation occurred |
| Bucket-local invalidation | writer mutates bucket `a` while cursor is on bucket `b` | bucket `b` cursor remains valid |
| Drop invalidation | cursor exists, then `dropBucket()` same bucket | next cursor operation returns `InvalidState` |
| Transaction closed | use cursor after commit or rollback | returns `TransactionClosed` |
| Accessors | cursor in no-item state | `key()` and `value()` return `null` without error |
| Accessors | cursor invalidated by write mutation | accessor fails with `InvalidState` |

## 9. Acceptance Criteria

This feature is complete for M5 when:

- `first()`, `last()`, `seek()`, `next()`, and `prev()` have one documented and
  tested behavior each
- cursor traversal works across single-leaf and multi-leaf trees
- the cursor stores enough root-to-leaf path state to step across leaf
  boundaries without restarting every move from the root
- no-item is an explicit normal state and remains distinct from lifecycle
  invalidation
- `seek()` positions at the first key `>= target` and returns no-item after the
  last key
- failed searches do not enable implicit recovery such as `seek(after_last)`
  then `prev()`
- `key()` and `value()` return `null` only for normal no-item state
- closed transactions fail cursor use with `TransactionClosed`
- same-bucket write mutations in a write transaction invalidate existing
  cursors conservatively with `InvalidState`
- bucket drop invalidates existing same-bucket cursors immediately
- bucket-local invalidation does not leak across unrelated buckets
- newly created writer cursors observe the latest staged bucket root after a
  mutation
- tests cover empty buckets, seek gaps, boundary moves, multi-leaf traversal,
  transaction closure, snapshot stability, and write-transaction invalidation
