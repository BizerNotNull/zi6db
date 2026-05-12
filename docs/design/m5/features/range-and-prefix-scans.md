# Range And Prefix Scans

## Goal

Define the concrete M5 implementation plan for ordered range scans and prefix scans on top of the bucket cursor API. This feature must preserve the M0-M4 storage, transaction, and copy-on-write rules while giving callers a predictable streaming iteration model in both forward and backward directions.

## Scope

This feature covers:

- A bound model for inclusive, exclusive, and unbounded endpoints.
- Forward and backward range positioning rules.
- Prefix scan planning, including lexicographic successor calculation.
- A mandatory `startsWith` guard for prefix iteration.
- Streaming semantics and cursor invalidation behavior.
- Tests and acceptance criteria specific to scan behavior.

This feature does not add custom comparators, materialized result sets, duplicate keys, or mutation-stable write cursors.

## Bound Model

### Public Shape

M5 should represent scan endpoints with a direction-independent bound type:

```zig
pub const Bound = union(enum) {
    unbounded,
    included: []const u8,
    excluded: []const u8,
};
```

Range scans should use:

```zig
pub const Range = struct {
    start: Bound = .unbounded,
    end: Bound = .unbounded,
};
```

If the API exposes separate forward and backward scan entry points, both must use the same `Bound` and `Range` model. Direction changes only the initial positioning and step function, not the meaning of the bounds.

### Comparison Rules

- All comparisons are raw byte-lexicographic comparisons.
- Empty byte strings are valid keys and valid bound payloads.
- Bounds do not apply text rules, Unicode collation, normalization, or terminator semantics.
- A key is in range only if it satisfies both `start` and `end`.

### Bound Predicates

The implementation should centralize bound checks in helpers equivalent to:

- `satisfies_start(key, .unbounded) = true`
- `satisfies_start(key, .included(k)) = key >= k`
- `satisfies_start(key, .excluded(k)) = key > k`
- `satisfies_end(key, .unbounded) = true`
- `satisfies_end(key, .included(k)) = key <= k`
- `satisfies_end(key, .excluded(k)) = key < k`

Using shared helpers matters because the initial positioning logic and the step-time guard logic must agree. M5 should reject any design where positioning uses one interpretation and per-item filtering uses another.

## Forward And Backward Ranges

### API Shape

The implementation may expose either:

```zig
pub fn scanRange(bucket: *Bucket, range: Range) !RangeCursor;
pub fn scanRangeReverse(bucket: *Bucket, range: Range) !RangeCursor;
```

or one range cursor with an explicit direction flag:

```zig
pub const ScanDirection = enum { forward, backward };
pub fn scanRange(bucket: *Bucket, range: Range, direction: ScanDirection) !RangeCursor;
```

The important requirement is semantic clarity, not the exact function spelling.

### Forward Positioning

Forward scans must establish the first candidate key as follows:

- `start = .unbounded`: position with `first()`.
- `start = .included(k)`: position with `seek(k)`.
- `start = .excluded(k)`: position with `seek(k)` and advance once if the positioned key equals `k`.

After initial positioning, the scan must verify the candidate also satisfies `end`. If it does not, the scan is empty.

### Backward Positioning

Backward scans must establish the first candidate key as follows:

- `end = .unbounded`: position with `last()`.
- `end = .included(k)`: position at the greatest key `<= k`.
- `end = .excluded(k)`: position at the greatest key `< k`.

This requires internal helpers such as `seek_le` and `seek_lt`, or equivalent logic built from tree descent. M5 should not depend on the public failed-`seek()` state for reverse positioning. In particular, it should not rely on undocumented behavior like "seek past end, then call `prev()`".

After initial positioning, the scan must verify the candidate also satisfies `start`. If it does not, the scan is empty.

### Step Rules

After yielding a key:

- Forward scans advance with `next()`.
- Backward scans advance with `prev()`.

Every step must re-check both bounds before yielding. This is required even if the starting position was valid, because moving across leaf boundaries must never leak an out-of-range key.

### Empty-Range Cases

The scan must yield no items when any of the following is true:

- The bucket is empty.
- Initial positioning finds no candidate.
- `start` sorts after `end`.
- `start` and `end` name the same byte string, and at least one side is exclusive.
- The first positioned candidate satisfies only one bound.

### Equality Examples

Over existing key `b`:

- `[b, b]` yields `b`.
- `(b, b)` yields nothing.
- `[b, b)` yields nothing.
- `(b, b]` yields nothing.

These cases should be enforced by normal predicate checks rather than bespoke one-off logic.

## Prefix Successor Rules

### Prefix Model

Prefix scans match keys where `std.mem.startsWith(u8, key, prefix)` is true. Prefix bytes are raw bytes, so embedded zero bytes and invalid UTF-8 are valid unless the underlying bucket key model rejects them globally.

### Planning Strategy

Prefix scanning should be implemented as a specialized range plan:

- Lower bound: `.included(prefix)`.
- Upper bound: the exclusive lexicographic successor of `prefix`, when one exists.
- Final guard: always verify `startsWith(prefix)` before yielding.

This keeps prefix scans aligned with the range engine while still handling successor edge cases safely.

### Successor Algorithm

The lexicographic successor of a prefix is the shortest byte string that sorts strictly after every key beginning with that prefix.

Recommended algorithm:

1. Copy the prefix bytes into a temporary buffer.
2. Walk backward from the last byte.
3. Find the rightmost byte that is not `0xff`.
4. Increment that byte by one.
5. Truncate the buffer immediately after that incremented byte.
6. Use the resulting byte slice as the exclusive upper bound.

Examples:

- `ab` -> `ac`
- `ab\xff` -> `ac`
- `ab\xfe\xff` -> `ab\xff`
- `\x00\xff` -> `\x01`

If every byte is `0xff`, there is no finite successor.

Examples:

- `\xff`
- `\xff\xff`

In those cases, the internal upper bound must be treated as unbounded.

### Required Safety Rules

- Successor calculation must not overflow.
- Successor calculation must not append bytes after truncation.
- Successor calculation must not mutate caller-owned memory.
- Prefix planning must work for empty prefixes.
- An empty prefix has no practical upper restriction and scans the whole bucket.

## `startsWith` Guard

The final prefix filter must always be a direct `startsWith(prefix)` check on each yielded key.

This guard is mandatory even when a successor upper bound exists because:

- Internal positioning bugs should fail closed rather than returning wrong keys.
- Successor-based stopping alone does not prove the current key still shares the prefix.
- Future refactors may change cursor stepping or reverse-scan entry logic.
- The all-`0xff` case has no finite successor and therefore depends entirely on the prefix check for termination.

Implementation rule:

- A candidate key that fails `startsWith(prefix)` terminates the prefix scan immediately.

The scan should not skip that key and continue, because once lexicographic order has advanced beyond the prefix region, no later key can re-enter it.

## Streaming Semantics

### No Materialization

Range and prefix scans must stream through the existing cursor machinery. M5 must not allocate or materialize the full result set before returning items.

Acceptable internal state includes:

- The underlying bucket cursor.
- The scan direction.
- Bound values or the computed prefix successor.
- A small amount of cursor-local bookkeeping such as `done` or `started`.

Unacceptable behavior includes:

- Copying all matching keys into an array before iteration.
- Reading the whole tree to pre-count results.
- Re-sorting results outside B+Tree order.

### Snapshot Behavior

- A scan created in a read transaction reads from that transaction's pinned metadata snapshot.
- A scan created after writes in a write transaction must observe that transaction's staged tree state.
- A scan from one transaction must never observe another transaction's uncommitted changes.

### Invalidation Behavior

M5 should reuse the same cursor invalidation policy as ordinary bucket cursors:

- Read-transaction scans remain valid until the transaction closes.
- Any `put()` or `delete()` in the same bucket invalidates write-transaction scans.
- `dropBucket()` invalidates scans for that bucket immediately.
- Transaction commit, rollback, or close invalidates all derived scans.

Once invalidated, scan operations should return the same `InvalidState` or `TransactionClosed` class used by the base cursor API. Normal end-of-range must remain distinguishable from lifecycle invalidation.

### End-Of-Scan Rules

- When the scan is exhausted, further step attempts remain exhausted.
- Exhaustion is not an error.
- Failing a bound check after movement transitions the scan to exhausted state.
- For prefix scans, failing `startsWith(prefix)` also transitions to exhausted state.

## Tests

The M5 test plan should add focused cases for scan setup, iteration, and boundary handling.

### Range Tests

- Inclusive forward range: `[b, d]` over `a..e` yields `b, c, d`.
- Exclusive forward range: `(b, d)` over `a..e` yields `c`.
- Mixed bounds: `(b, d]` and `[b, d)` yield the expected subsets.
- Unbounded start: `[..c]` yields keys up to `c` according to end inclusivity.
- Unbounded end: `[c..]` yields keys from `c` onward according to start inclusivity.
- Equal inclusive bounds with existing key yield exactly one item.
- Equal bounds with any exclusive side yield no items.
- Start greater than end yields no items.
- Empty bucket range yields no items.
- Empty byte-string key participates in range comparisons normally.
- Forward multi-leaf range yields each in-range key exactly once across page boundaries.
- Backward inclusive range: reverse `[b, d]` over `a..e` yields `d, c, b`.
- Backward exclusive upper bound: reverse `[b, d)` over `a..e` yields `c, b`.
- Backward unbounded end starts from the largest visible key.
- Reverse scans do not depend on failed `seek()` recovery behavior.

### Prefix Tests

- Prefix present: `ab` over `aa, ab, aba, ac` yields `ab, aba`.
- Prefix missing yields no items.
- Empty prefix yields the whole bucket.
- Embedded-zero prefix matches raw-byte keys correctly.
- Prefix with finite successor stops before the first non-matching key.
- Prefix ending in one or more `0xff` bytes computes the correct successor.
- All-`0xff` prefix uses unbounded upper logic and still stops by `startsWith`.
- False-positive successor guard test proves `startsWith` prevents non-prefix leakage.
- Multi-leaf prefix scan yields matching keys exactly once across page boundaries.

### Streaming And Lifecycle Tests

- Range scans do not allocate proportional to result size.
- Prefix scans do not materialize full results.
- Reader scan keeps a stable snapshot while a concurrent writer commits.
- New write-transaction scan observes staged writes from the same transaction.
- Write-transaction scan is invalidated by same-bucket `put()`.
- Write-transaction scan is invalidated by same-bucket `delete()`.
- Scan is invalidated by `dropBucket()`.
- Scan use after transaction close returns lifecycle error, not empty iteration.

## Acceptance

- The feature uses a single bound model for inclusive, exclusive, and unbounded endpoints.
- Forward scans begin from the first key satisfying `start` and never yield a key past `end`.
- Backward scans begin from the last key satisfying `end` and never yield a key before `start`.
- Range scans enforce both bounds on every yielded item.
- Empty, equal-bound, and inverted-bound ranges behave deterministically.
- Prefix scans use raw-byte `startsWith` semantics.
- Prefix successor calculation is correct for normal, mixed, and high-byte prefixes.
- Prefix scans always apply the final `startsWith` guard, including when a finite successor exists.
- All-`0xff` prefixes work without overflow or out-of-range leakage.
- Range and prefix scans stream through cursor movement without materializing full result sets.
- Scan behavior respects transaction snapshots and cursor invalidation rules already defined for M5.
- The test suite covers forward and backward ranges, prefix edge cases, streaming behavior, and lifecycle errors.
