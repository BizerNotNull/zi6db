# M3 Transaction MVP Implementation Plan

This plan is based on `M3: Transaction MVP` in `MILESTONES.md`, the product direction in `ROADMAP.md`, the repository rules in `AGENTS.md`, and the frozen M0 design documents under `docs/design/m0/`.

M3 introduces real transaction behavior while preserving the M0 copy-on-write, metadata-switch, dirty-page ownership, and commit-ordering model required by M4 crash safety. M3 should make transactions correct in-process first; it must not introduce an API or page lifecycle that M4 would need to replace.

## 1. Goals

- Add read-only and read-write transaction APIs.
- Allow multiple concurrent read transactions inside one process.
- Enforce at most one active write transaction.
- Give each read transaction a stable metadata snapshot.
- Keep uncommitted write changes invisible to other transactions.
- Make rollback discard all private write state without repairing disk.
- Route write commit through the M0-compatible copy-on-write commit pipeline.
- Preserve dirty page ownership and freelist pending/reusable semantics needed by M4 and M6.
- Make the M3 transaction API the only mutation boundary; any M2 convenience API that remains must be a thin transaction wrapper.
- Provide tests for snapshot visibility, writer exclusivity, rollback, lifecycle errors, and M4 compatibility boundaries.

## 2. Inputs And Frozen Constraints

M3 must follow these M0 decisions:

- The visible database state is selected by metadata, not by scanning the file.
- `generation` is the startup ordering field; `txn_id` is the visibility and freelist cutoff field.
- Committed write transactions increment both `generation` and `txn_id`.
- Read transactions pin the selected metadata snapshot at begin time and never rebase.
- Write transactions start from the current selected metadata snapshot.
- Visible pages are never modified in place.
- Only the active writer may own dirty pages.
- Rollback only discards in-memory transaction state.
- A commit writes branch, leaf, overflow, and freelist pages before writing the inactive metadata slot.
- M4 recovery will select the latest valid metadata slot; M3 must keep that slot-switch shape.
- Deleted pages enter `pending_free[freeing_txn_id]` before they can become reusable.
- Pages in `pending_free` may move to `reusable` only after all older read transactions have ended.
- `close()` must fail with `InvalidState` while transactions are active.

## 3. Public Transaction API

Implement the M0 lifecycle shape without over-expanding the future bucket/cursor API.

```zig
pub fn beginRead(db: *DB) !ReadTx;
pub fn beginWrite(db: *DB) !WriteTx;

pub fn id(tx: *const Tx) u64;
pub fn isReadOnly(tx: *const Tx) bool;
pub fn rollback(tx: *Tx) void;
pub fn commit(tx: *WriteTx) !void;
```

For the M2 single-key-space engine, expose KV operations through transactions instead of allowing direct unscoped writes:

```zig
pub fn get(tx: *const Tx, allocator: Allocator, key: []const u8) !?[]u8;
pub fn exists(tx: *const Tx, key: []const u8) !bool;
pub fn put(tx: *WriteTx, key: []const u8, value: []const u8) !void;
pub fn delete(tx: *WriteTx, key: []const u8) !bool;
```

API rules:

- `beginRead()` fails with `InvalidState` if the database is closing or closed.
- `beginRead()` fails with `InvalidState` or another documented lifecycle error if `max_read_transactions` would be exceeded.
- `beginWrite()` fails with `ReadOnlyTransaction` when the database was opened read-only.
- `beginWrite()` fails with `WriteConflict` when a writer is already active.
- Mutating through a read transaction fails with `ReadOnlyTransaction`.
- Reusing a committed, rolled-back, or failed transaction fails with `TransactionClosed`.
- `commit()` is only valid on `WriteTx`.
- `rollback()` is valid on both read and write transactions and is idempotent only if the implementation explicitly documents that choice; the conservative default is that second use reports `TransactionClosed` on later operations.
- `commit()` success closes the write transaction.
- `commit()` failure closes the write transaction into a terminal failed state.
- `close()` returns `InvalidState` if any read or write transaction remains active.
- `get()` must return caller-owned memory, matching the M2 safe ownership model. M3 should not expose borrowed page-cache slices until cursor/page lifetime rules are introduced later.
- Key and value input slices are copied into dirty pages before the mutating call returns; caller buffers may be reused immediately after `put()`.
- Direct M2 `DB.get` and `DB.exists` helpers, if kept for compatibility, must internally create a short read transaction and return the same owned-value semantics.
- Direct M2 `DB.put` and `DB.delete` helpers, if kept for compatibility, must internally create a write transaction, route all mutation through `WriteTx`, commit or roll back on error, and must not bypass dirty-page ownership.
- Transaction-bound methods are the canonical M3 API. New examples and tests should prefer explicit `beginRead()` and `beginWrite()` usage.

## 4. Transaction State Model

Use an explicit state machine so lifecycle errors are easy to test.

| State | Applies to | Meaning | Allowed next states |
| --- | --- | --- | --- |
| `active_read` | `ReadTx` | Snapshot is pinned and readable | `closed_rollback` |
| `active_write` | `WriteTx` | Writer owns private dirty state | `closed_commit`, `closed_rollback`, `closed_failed` |
| `committing` | `WriteTx` | Commit pipeline is running | `closed_commit`, `closed_failed` |
| `closed_commit` | `WriteTx` | Commit succeeded | none |
| `closed_rollback` | both | Rollback completed | none |
| `closed_failed` | `WriteTx` | Commit failed and result may be ambiguous after metadata write begins | none |

Each transaction stores:

- `db` pointer or handle.
- `mode`: read-only or read-write.
- `state`.
- `selected_slot` copied from the metadata chosen at transaction begin.
- `snapshot_generation`.
- `snapshot_txn_id`.
- `snapshot_root_page_id`.
- `snapshot_freelist_page_id`.
- `snapshot_page_count`.
- A stable reference or copy of the metadata selected at begin time.
- An active-reader registry token for read transactions, so cleanup removes exactly the entry that was registered.

Each write transaction additionally stores:

- `base_metadata`.
- `base_selected_slot`.
- `dirty_pages`: private map from target page id to page buffer.
- `copied_pages`: map from old visible page id to cloned page id.
- `new_pages`: page ids allocated by this transaction.
- `abandoned_new_pages`: private allocations that were never written or published, if the allocator supports reusing them inside the same transaction.
- `pending_free_delta`: pages removed by this transaction, grouped under the future `new_txn_id`.
- `new_root_page_id` after staged mutations.
- `new_freelist_page_id` after finalizing freelist state.
- `new_page_count`.
- `commit_slot`: inactive metadata slot chosen before metadata write.

Implementation notes:

- `snapshot_*` fields are immutable after transaction begin.
- `selected_slot` and `base_selected_slot` are used to enforce strict A/B alternation and to test that the writer never rebases during commit.
- `new_page_count` must be the exclusive upper bound of all page ids that will be considered allocated by the new metadata. If M3 allocates a page id and later abandons it, the transaction must either reuse that id for another private dirty page before commit or keep it outside the committed `page_count`; it must not publish gaps that masquerade as reusable pages.
- Terminal states must be observable in tests: committed, rolled-back, and failed transactions all reject later reads, writes, and commit attempts with `TransactionClosed`. Because the M0 sketch makes `rollback()` return `void`, M3 must explicitly document whether a second rollback is a no-op or a debug/lifecycle assertion.

## 5. Read Snapshot Semantics

`beginRead()` must:

1. Acquire the database metadata/read-registry lock briefly.
2. Copy or pin the current in-memory selected metadata.
3. Record `generation`, `txn_id`, root page id, freelist page id, and page count.
4. Register the reader's `snapshot_txn_id` in the in-process reader registry before the metadata/read-registry lock is released.
5. Release the metadata/read-registry lock.

Read operations must:

- Traverse from the transaction's pinned root page id.
- Reject page ids outside the transaction's pinned `page_count`.
- Read only committed pages reachable from the pinned root.
- Use the transaction's pinned freelist page for any freelist-aware validation or future stats path; never switch to a newer freelist page while the transaction is active.
- Never consult the writer's dirty page map.
- Never observe metadata published after the read transaction began.
- Return copies of values into the caller-provided allocator, not borrowed references to storage buffers.
- Treat checksum, page type, page id, or routing failures in the pinned snapshot as `Corruption`; a later successful commit must not mask corruption visible to the reader's own snapshot.

`rollback(ReadTx)` must:

1. Mark the transaction closed.
2. Remove its `snapshot_txn_id` from the reader registry.
3. Trigger or allow pending-free promotion checks for future writers.

Reader registry rules:

- Metadata capture and reader registration are one critical section. Otherwise, a writer could incorrectly promote pending-free pages while a reader is beginning on the older snapshot.
- The registry must track enough identity to remove one reader without disturbing other readers on the same `snapshot_txn_id`.
- Closing a read transaction must remove the registry entry exactly once, including when later operations fail with `Corruption` or `IoError`.
- If `max_read_transactions` is configured, the limit applies before the reader is returned to the caller.

Snapshot tests must prove that a reader that began before a successful write commit continues to observe the old value, while a new reader observes the committed value.

## 6. Single Writer Semantics

Use one in-process writer gate owned by `DB`.

`beginWrite()` must:

1. Reject read-only database handles.
2. Atomically acquire the writer gate.
3. Capture the current selected metadata as the writer base snapshot.
4. Initialize an empty dirty-page owner for the transaction.
5. Register the transaction as active.

Rules:

- A write transaction may coexist with read transactions.
- A second writer must fail fast with `WriteConflict`; it must not block indefinitely in M3.
- The writer gate is released only by commit success, rollback, or terminal commit failure cleanup.
- No helper below the transaction layer may create dirty pages unless it is passed the active `WriteTx` owner.
- The writer's base metadata must remain the selected in-memory metadata until commit publishes a replacement. Because M3 allows only one writer, a mismatch means an implementation bug or invalid lifecycle state, not a reason to rebase the transaction silently.
- Commit must write the inactive metadata slot opposite `base_selected_slot`; it must not choose a slot by reading the file again during commit.
- The writer gate must cover the entire write transaction lifetime, not only the final commit call, so dirty-page ownership cannot be split across writers.

## 7. Dirty Page Ownership

Dirty page ownership is the core M3/M4 compatibility boundary.

Only the active `WriteTx` may own:

- Cloned branch pages.
- Cloned leaf pages.
- Newly allocated branch or leaf pages.
- Future overflow pages once M6 enables them.
- The new freelist page.
- The inactive metadata page contents before writeout.

Mutation rules:

- Before modifying a visible page, clone it to a page id that is not visible from any current metadata root.
- Mutate only the clone.
- If the same old page is modified again in the same transaction, reuse the existing clone from `copied_pages`.
- Newly split pages are born directly in `dirty_pages`.
- Root changes update only `WriteTx.new_root_page_id` until commit.
- Deleted visible pages are added to `pending_free_delta`; they are not added directly to reusable pages.
- Pages allocated and then abandoned within the same uncommitted transaction may be discarded from the transaction's private state, but must not be published to the persistent freelist.
- Dirty normal pages must carry the pending `new_generation` in their `page_generation` field before checksums are computed.
- The inactive metadata page is staged separately from normal dirty pages and is not visible through the page cache or read path before the metadata write step.
- If allocation uses the persistent reusable set, the allocator must first apply the reader-safety cutoff. Otherwise M3 should prefer tail allocation over speculative reuse.

Ownership invariants:

- Read transactions must not hold references to dirty page buffers.
- Dirty page buffers must be unreachable from `DB.current_metadata` until commit switches metadata.
- Dirty page ids must not overlap page ids reachable from any active reader snapshot. Reusing a page id is allowed only when the freelist rule proves no active reader can still reach the old page version.
- A dirty page checksum/page generation field should be finalized before the non-metadata write phase, even if M3 does not yet provide full M4 fault injection.
- Dirty pages from a failed or rolled-back writer must not remain in shared caches where later readers or writers can accidentally observe them.
- Any internal page cache must key dirty entries by transaction ownership or keep them outside the shared committed-page cache until after commit success.

## 8. Commit Semantics

M3 commit should use the M0 ordering even if M4 later strengthens sync and recovery validation.

Commit steps:

1. Transition `active_write -> committing`.
2. Allocate `new_txn_id = base.txn_id + 1`.
3. Allocate `new_generation = base.generation + 1`.
4. Promote eligible old `pending_free` groups to reusable using the current oldest active reader txn id.
5. Apply the transaction's `pending_free_delta` as `pending_free[new_txn_id]`.
6. Finalize dirty branch and leaf pages, including page headers and checksums.
7. Build and finalize the new freelist page.
8. Write all non-metadata dirty pages to page ids not visible from the base metadata.
9. Call the durability hook for the pre-metadata data boundary.
10. Build inactive metadata with the new root, new freelist, `new_txn_id`, `new_generation`, and new page count.
11. Compute the metadata checksum.
12. Write the inactive metadata slot opposite `base_selected_slot`.
13. Call the durability hook for the metadata boundary.
14. Atomically publish the new metadata in memory.
15. Clear private dirty state.
16. Release the writer gate.
17. Mark the transaction `closed_commit`.

M3 may initially implement the durability hooks as the existing storage sync behavior available after M1/M2, but the hook boundaries and error propagation must already match M0. If a hook or write fails before step 14, in-memory current metadata must not switch.

Before step 10, commit must assert that the in-memory selected metadata still matches `base_metadata.generation`, `base_metadata.txn_id`, and `base_selected_slot`. A mismatch is `InvalidState` or an internal assertion failure; M3 must not silently rebase the writer onto newer metadata.

Commit success means:

- New transactions in the same process see the new root.
- Older read transactions continue using their old root.
- The write transaction is closed.
- Dirty pages are no longer owned by the transaction.

Commit failure means:

- The write transaction enters `closed_failed`.
- The writer gate is released after cleanup.
- In-memory current metadata does not switch unless step 14 completed. If step 14 completed, `commit()` must return success, not failure.
- The caller may not reuse the transaction.
- If failure occurred after metadata write began but before metadata sync was confirmed, the persisted result is ambiguous and M4 recovery will be authoritative.

Failure classes:

- Class A: failure before metadata write begins, including allocation, normal page write, or pre-metadata sync failure. Return `IoError` or the specific mapped storage error, leave in-memory metadata unchanged, and expect recovery to select the old metadata.
- Class B: metadata write may have begun, but metadata sync was not confirmed. Return `IoError`, leave in-memory metadata unchanged, mark the writer terminal, and document that reopen may select old or new metadata depending on which slot validates.
- Class C: metadata sync confirmed. The commit may publish metadata in memory and return success. If later in-memory cleanup fails, it must not be reported as commit failure that contradicts the durable success point.

M3 test hooks should be able to fail each of these boundaries independently without requiring power-loss simulation. M4 will add crash/reopen verification for the same boundaries.

## 9. Rollback Semantics

Rollback must not perform disk repair.

`rollback(WriteTx)` must:

1. Mark the transaction closing to reject further mutation.
2. Drop dirty branch and leaf page buffers.
3. Drop copied-page mappings.
4. Drop newly allocated page ownership.
5. Drop `pending_free_delta`.
6. Drop staged root and freelist changes.
7. Release the writer gate.
8. Mark the transaction `closed_rollback`.

Rollback invariants:

- `DB.current_metadata` is unchanged.
- Existing readers are unaffected.
- New readers see the same state they would have seen before the write transaction began.
- No page freed by the rolled-back transaction becomes reusable because the delete never committed.
- Rollback performs no page writes, metadata writes, file truncation, syncs, or recovery scans.
- Rollback is valid only before commit starts. Once the state is `committing` or `closed_failed`, the transaction is terminal and the caller cannot use rollback to make the persisted result less ambiguous.
- Any private reusable-page reservations made by the writer return only to the in-memory allocator view for future transactions; they must not be persisted unless a later transaction commits a freelist page.

Rollback tests must include insert rollback, update rollback, delete rollback, root split rollback, and rollback while older readers are active.

## 10. Freelist And Reader Registry

M3 must introduce enough reader tracking to make page reuse safe.

`DB` should maintain:

- Active reader count.
- Active reader txn ids, or an equivalent min-tracking structure.
- `oldest_reader_txn_id()` for freelist reuse decisions.
- Active writer flag/owner.

Promotion rule:

- A pending-free group with `freeing_txn_id = X` can move to reusable only when no active reader has `snapshot_txn_id < X`.

Practical M3 rule:

- Run pending-free promotion at the start of a write transaction or during commit finalization.
- Do not require a background task.
- Do not infer reusable pages by scanning unreachable pages.
- Do not reuse pages freed by the same write transaction.
- Promotion must read from the selected freelist snapshot captured by the writer, then publish the result only by writing a new freelist page in the same commit.
- Pending-free groups that are not yet eligible must be preserved with their original `freeing_txn_id`; M3 must not flatten them into reusable pages or drop their grouping.
- If there are no active readers, pending-free groups already present in the writer's base freelist with `freeing_txn_id <= base.txn_id` are eligible for promotion. The current transaction's `pending_free_delta` is not reusable by the same transaction.
- A read transaction pins its metadata and root, but it also protects all pages that could be reachable from that metadata. The allocator must treat the reader registry as authoritative, not the current root alone.
- Restart behavior remains an open/recovery concern: after process restart there are no surviving in-process readers, but M3 commit code must still persist pending-free state exactly so M4/M6 can make the correct promotion decision.

## 11. Execution Order

Implement M3 in this order to reduce redesign risk:

1. Add transaction structs, state enum, lifecycle validation, and active transaction accounting.
2. Move M2 direct KV operations behind transaction-bound read/write methods, keeping any compatibility helpers as transaction wrappers only.
3. Add read snapshot capture from selected metadata and reader registry tracking.
4. Add the single-writer gate and write transaction base snapshot capture.
5. Refactor B+Tree mutation helpers to require `WriteTx` and produce dirty pages instead of in-place visible-page edits.
6. Add page clone/allocation helpers with `copied_pages`, `dirty_pages`, `new_pages`, and `pending_free_delta`.
7. Add owned-value read tests so `get()` results remain valid after the transaction closes and do not expose dirty buffers.
8. Add rollback cleanup and lifecycle tests before enabling commit.
9. Implement the M0-shaped commit pipeline with durability hook placeholders at normal-page write, pre-metadata sync, metadata write, and metadata sync boundaries.
10. Add in-memory metadata publication only after the metadata write and metadata sync boundary succeeds for the selected durability mode.
11. Add snapshot visibility, writer exclusivity, rollback, and transaction lifecycle tests.
12. Add M4 compatibility tests that assert commit ordering through injected storage/durability hooks.
13. Update only API-facing docs or examples in the same PR if M3 implementation changes public usage; do not change M0 format rules.

## 12. Test Matrix

| Area | Scenario | Expected result |
| --- | --- | --- |
| Read snapshot | Reader starts, writer updates key and commits, old reader reads key | Old reader sees old value |
| Read snapshot | New reader starts after writer commit | New reader sees new value |
| Read snapshot | Reader scans while writer splits pages | Reader traversal remains stable |
| Read snapshot | Reader uses pinned page count after file grows | Reader rejects pages outside its snapshot |
| Read snapshot | Reader hits corruption in its pinned root | Returns `Corruption`; newer metadata does not mask it |
| Read value ownership | `get()` value is kept after read tx rollback | Caller-owned value remains valid |
| Read registry | Exceed configured `max_read_transactions` | Begin read fails without leaking registry state |
| Writer exclusivity | Begin second writer while first is active | `WriteConflict` |
| Writer lifecycle | Begin writer while readers are active | Succeeds |
| Writer lifecycle | Writer base metadata mismatches current metadata before commit | Commit fails as lifecycle/internal error; no silent rebase |
| Read-only database | Begin writer on read-only DB | `ReadOnlyTransaction` |
| Mutation guard | Call `put` or `delete` through read tx | `ReadOnlyTransaction` |
| M2 compatibility | Direct `DB.put` helper if retained | Internally uses write transaction and commit hooks |
| M2 compatibility | Direct `DB.get` helper if retained | Internally uses read transaction and owned-value return |
| Commit lifecycle | Reuse committed writer | `TransactionClosed` |
| Rollback lifecycle | Reuse rolled-back tx | `TransactionClosed` |
| Failed lifecycle | Reuse failed writer after injected commit error | `TransactionClosed` |
| Close lifecycle | Close DB with active reader | `InvalidState` |
| Close lifecycle | Close DB with active writer | `InvalidState` |
| Rollback insert | Insert new key then rollback | Key is absent for new reader |
| Rollback update | Update existing key then rollback | Old value remains visible |
| Rollback delete | Delete existing key then rollback | Key remains visible |
| Rollback split | Force page/root split then rollback | Tree root and lookups remain unchanged |
| Rollback I/O | Rollback dirty writer | No page write, metadata write, sync, or truncate hook is called |
| Dirty isolation | Writer updates key before commit | Concurrent reader cannot see update |
| Dirty ownership | Mutating helper called without `WriteTx` | Compile-time or runtime rejection |
| Dirty ownership | Dirty page enters shared cache before commit | Test detects and rejects visibility leak |
| Dirty ownership | Dirty normal page finalized | `page_generation` equals pending `new_generation` before checksum |
| Allocation | Transaction abandons a newly allocated page | Page is reused privately or remains outside committed `page_count` |
| Page reuse | Delete page while older reader active | Freed page stays pending, not reusable |
| Page reuse | Older reader closes before later writer | Eligible pending pages can become reusable |
| Commit order | Inject failure before non-metadata write | Metadata does not switch |
| Commit order | Inject failure before metadata write | Metadata does not switch |
| Commit order | Inject failure during metadata write | Transaction closes failed; in-memory metadata does not switch |
| Commit order | Inject metadata sync failure | Transaction closes failed; persisted outcome documented as ambiguous |
| Commit order | Successful metadata boundary | In-memory metadata switches after boundary only |
| Generation | Successful commit | `generation` and `txn_id` increment by one |
| Slot switch | Commit from metadata A | Writes inactive metadata B |
| Slot switch | Next commit from metadata B | Writes inactive metadata A |

Minimum executable test groups:

- `transaction_lifecycle`.
- `snapshot_visibility`.
- `single_writer`.
- `rollback_correctness`.
- `dirty_page_ownership`.
- `freelist_reader_safety`.
- `commit_ordering_hooks`.
- `m2_compatibility_wrappers`, if direct DB-level helpers remain public.

## 13. M4 Prerequisite Compatibility

M3 must leave these seams ready for M4:

- A storage/durability interface with explicit non-metadata write, pre-metadata sync, metadata write, and metadata sync steps.
- Fault-injection hooks at every M0 crash matrix boundary.
- Metadata A/B slot selection already implemented.
- Equal-generation metadata must remain impossible during normal commit and must remain a `Corruption` condition for open/recovery.
- Page checksums finalized before pages are written.
- Metadata checksums finalized before metadata write.
- Normal-page `page_generation` assigned from the pending commit generation before checksum calculation.
- `generation` and `txn_id` monotonicity enforced on every successful commit.
- Dirty pages written only to ids outside the currently visible state.
- In-memory metadata switched only after the metadata boundary reports success.
- Commit failure after metadata write treated as terminal and potentially ambiguous.
- Startup open/recovery remains responsible for authoritative persisted state after ambiguous failure.
- Freelist persists `pending_free` groups rather than flattening all freed pages into reusable.
- Reader registry exposes the oldest pinned transaction id for safe pending-free promotion.
- Failed commits may leave unreachable tail pages on disk; M3 must not classify those pages as corruption, auto-reuse them, or add them to the freelist without a later committed freelist update.
- The commit API must preserve the M0 durability-mode contract: `strict` and `normal` success mean the selected mode's sync requirements succeeded, while `unsafe` may skip flushes but still must not expose in-process partial state before commit returns.

M3 should not add shortcuts such as in-place overwrite, direct root pointer mutation, one metadata page only, or untracked page reuse. Any such shortcut would force M4 to replace transaction semantics instead of only strengthening durability and recovery.

## 14. Non-Goals

M3 does not implement:

- Full crash-safe recovery validation; that is M4.
- New on-disk format decisions; M0 is authoritative.
- Multi-writer commits.
- Cross-process shared/exclusive file locking; M7 extends this.
- Bucket/table namespaces; M5 owns that API expansion.
- Cursor, range scan, or prefix scan APIs; M5 owns them.
- Overflow page large-value support; M6 owns it, though M3 must preserve page ownership semantics for it.
- Full compaction, vacuum, or operational backup.
- Performance tuning beyond avoiding obvious transaction-state bottlenecks.

## 15. Acceptance Checklist

M3 is complete when:

- `beginRead()` returns a transaction pinned to a stable metadata snapshot.
- Reader registration is atomic with metadata snapshot capture and respects `max_read_transactions`.
- Multiple readers can be active at the same time.
- A writer can run while readers are active.
- A second writer fails with `WriteConflict`.
- Any retained direct M2 DB-level helpers route through transactions and cannot bypass lifecycle, snapshot, or dirty-page ownership rules.
- `get()` returns caller-owned values and does not expose borrowed dirty or committed page buffers.
- Read transactions never observe uncommitted writer dirty pages.
- Older read transactions keep seeing their original snapshot after writer commit.
- New read transactions see the latest committed metadata after writer commit.
- Write mutations use copy-on-write and never modify visible pages in place.
- Dirty pages are owned only by the active write transaction.
- Dirty normal pages carry the pending commit generation before checksums are finalized.
- Allocated-but-abandoned private pages are not published as committed page-count gaps or persistent freelist entries.
- Rollback of inserts, updates, deletes, and split-producing writes leaves no visible partial changes.
- Rollback performs no disk repair, metadata write, sync, or file truncation.
- Commit increments `generation` and `txn_id` exactly once.
- Commit writes the inactive metadata slot according to the A/B switch rule.
- In-memory metadata switches only after the metadata boundary succeeds.
- Commit failure leaves the writer terminal and unreusable.
- Metadata-sync failure is treated as ambiguous on disk but does not switch in-memory metadata.
- `close()` rejects active transactions.
- Freelist pending pages are not reused while older readers are active.
- Tests cover snapshot visibility, writer exclusivity, rollback correctness, lifecycle errors, dirty page ownership, page reuse safety, M2 compatibility wrappers, and commit ordering hooks.
- The implementation can add M4 strict sync, recovery selection, and crash fault injection without replacing the transaction API or dirty-page lifecycle.
