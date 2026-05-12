# Transaction API And State Machine

This document defines the implementation plan for the M3 feature `transaction API and state machine`. It refines the repository-wide M3 plan in [m3-implementation-plan.md](/D:/代码D/zi6db/docs/design/m3/m3-implementation-plan.md) and must remain consistent with the frozen lifecycle and error semantics in [api-sketch.md](/D:/代码D/zi6db/docs/design/m0/api-sketch.md).

## 1. Scope

M3 must introduce the first concrete transaction lifecycle used by all mutable database operations. The feature covers:

- `beginRead(db: *DB) !ReadTx`
- `beginWrite(db: *DB) !WriteTx`
- `commit(tx: *WriteTx) !void`
- `rollback(tx: *Tx) void`
- the transaction state enum and lifecycle guards
- closed and failed transaction semantics
- DB-level helper wrappers that route legacy convenience APIs through transactions
- public error behavior and internal failure mapping
- executable tests and acceptance criteria

This feature does not define new on-disk formats. It provides the in-process API and state model that later milestones will harden for crash recovery, freelist reuse, and richer data access.

## 2. Frozen Inputs

The implementation must preserve these M0 constraints:

- `beginRead()` pins the current metadata snapshot.
- `beginWrite()` allows at most one active writer.
- `commit()` success closes the write transaction.
- `commit()` failure makes the write transaction terminal.
- `rollback()` closes read and write transactions.
- `close()` must fail with `InvalidState` when transactions are still active.
- `ReadOnlyTransaction`, `WriteConflict`, `TransactionClosed`, `InvalidState`, and `IoError` keep their M0 meanings.

The feature must also follow the M3 repository plan:

- multiple read transactions may coexist
- one write transaction may coexist with readers
- uncommitted writes remain private to the writer
- rollback discards only private transaction state
- direct DB mutation helpers, if retained, become transaction wrappers only

## 3. API Surface

### 3.1 Canonical Transaction Entry Points

The canonical M3 API is:

```zig
pub fn beginRead(db: *DB) !ReadTx;
pub fn beginWrite(db: *DB) !WriteTx;

pub fn id(tx: *const Tx) u64;
pub fn isReadOnly(tx: *const Tx) bool;
pub fn commit(tx: *WriteTx) !void;
pub fn rollback(tx: *Tx) void;
```

Implementation rules:

- `beginRead()` captures the current selected metadata and registers the reader before returning.
- `beginWrite()` acquires the single-writer gate and captures the current selected metadata as the writer base snapshot.
- `commit()` exists only on `WriteTx`.
- `rollback()` works on both `ReadTx` and `WriteTx`.
- all transaction-bound read or write methods must validate state before touching tree or storage helpers.

### 3.2 Transaction Method Expectations

M3 should expect all data access to hang off transactions, even if some KV methods remain in temporary M2-compatible form. At minimum:

- read paths accept `*const Tx` or `*const ReadTx`
- mutating paths accept `*WriteTx`
- no internal helper may create dirty pages without an owning `WriteTx`

## 4. State Enum And Lifecycle

### 4.1 Required State Enum

Use an explicit transaction state enum:

```zig
pub const TxState = enum {
    active_read,
    active_write,
    committing,
    closed_commit,
    closed_rollback,
    closed_failed,
};
```

Expected usage:

- `active_read` applies to live `ReadTx`
- `active_write` applies to live `WriteTx` before commit starts
- `committing` is a transient write-only state that blocks rollback and further mutation
- `closed_commit`, `closed_rollback`, and `closed_failed` are terminal states

### 4.2 Allowed Transitions

| Current state | Operation | Next state |
| --- | --- | --- |
| `active_read` | `rollback()` | `closed_rollback` |
| `active_write` | `rollback()` | `closed_rollback` |
| `active_write` | `commit()` start | `committing` |
| `committing` | commit success | `closed_commit` |
| `committing` | commit failure | `closed_failed` |
| terminal state | any later operation | no transition; reject |

Lifecycle rules:

- there is no upgrade path from `active_read` to `active_write`
- there is no reopen path from any closed state
- `rollback()` must not rescue a transaction once it has entered `committing`
- terminal states must be observable through later API errors even if internal cleanup already ran

## 5. Closed And Failed Semantics

### 5.1 Closed Semantics

A transaction is considered closed after:

- successful `commit()`
- successful `rollback()`
- failed `commit()`

Once closed:

- any later read method must fail with `TransactionClosed`
- any later write method must fail with `TransactionClosed`
- a second `commit()` attempt must fail with `TransactionClosed`
- DB close accounting must treat the transaction as inactive

### 5.2 Failed Semantics

`closed_failed` is required for write transactions whose commit did not complete successfully.

Rules:

- `closed_failed` is terminal and externally indistinguishable from other closed states except for debugging and targeted tests
- the writer gate must already be released before the transaction becomes externally reusable by the DB
- in-memory selected metadata must not switch on a failed commit
- private dirty state must be detached so later readers or writers cannot observe it
- a failed commit does not imply that the durable result is absent; reopen and recovery remain authoritative for ambiguous storage failures

## 6. DB Helper Wrappers

M3 should make explicit transactions the primary API, but temporary DB-level wrappers may remain for compatibility.

### 6.1 Read Helpers

If `DB.get` or `DB.exists` remains public, each helper must:

1. call `beginRead()`
2. run the transaction-bound read operation
3. `rollback()` the read transaction before returning
4. preserve the same owned-value semantics as the transaction API

Requirements:

- helpers must not bypass snapshot capture
- helpers must not borrow storage-backed memory into the caller
- helper cleanup must happen even on read-path failure

### 6.2 Write Helpers

If `DB.put` or `DB.delete` remains public, each helper must:

1. call `beginWrite()`
2. run the transaction-bound mutating operation
3. call `commit()` on success
4. call `rollback()` on any pre-commit failure

Requirements:

- helpers must not mutate storage outside a `WriteTx`
- helpers must propagate `WriteConflict`, `ReadOnlyTransaction`, `InvalidState`, and `IoError` without remapping them to generic failures
- helpers must not swallow commit ambiguity; a commit failure still fails the helper call

## 7. Error Plan

M3 must expose only the M0 public error model while allowing more specific internal checks.

### 7.1 Begin Errors

- `beginRead()` returns `InvalidState` when the DB is closing, closed, or cannot admit another reader under `max_read_transactions`
- `beginWrite()` returns `ReadOnlyTransaction` when the DB handle is read-only
- `beginWrite()` returns `WriteConflict` when another writer is active
- either begin path may return `IoError` or `Corruption` if snapshot acquisition or required metadata validation fails

### 7.2 Operation Errors

- calling a mutating API through a read transaction returns `ReadOnlyTransaction`
- calling any API on a terminal transaction returns `TransactionClosed`
- calling `commit()` on a transaction in the wrong lifecycle phase returns `TransactionClosed` or `InvalidState`, but the implementation should pick one rule and test it consistently
- snapshot validation failures discovered during reads return `Corruption`
- storage read, write, or sync failures return `IoError`

### 7.3 Close Errors

- `close(db)` returns `InvalidState` while any read or write transaction remains active

## 8. Implementation Slices

Implement this feature in narrow slices so lifecycle behavior becomes testable before full mutation logic lands.

### 8.1 Slice 1: Transaction Structs And Accounting

- add `Tx`, `ReadTx`, and `WriteTx` structs
- add `TxState`
- add DB counters or registries for active readers and active writer ownership
- add state validation helpers used by every transaction-bound method

### 8.2 Slice 2: Begin And End Lifecycle

- implement `beginRead()` snapshot capture and reader registration
- implement `beginWrite()` writer gate acquisition and base snapshot capture
- implement `rollback()` cleanup for both read and write transactions
- make `close()` reject active transactions

### 8.3 Slice 3: Commit State Machine

- add `active_write -> committing` transition
- add commit cleanup paths for success and failure
- ensure any failed commit lands in `closed_failed`
- ensure the writer gate and private state are always released exactly once

### 8.4 Slice 4: DB Wrapper Routing

- move retained DB-level helpers onto transaction wrappers
- remove any direct mutation path that bypasses `WriteTx`
- keep examples and tests focused on explicit transactions first

### 8.5 Slice 5: Error And Invariant Hardening

- centralize lifecycle error mapping
- assert illegal state transitions in debug builds
- add targeted tests for terminal-state reuse, helper cleanup, and close rejection

## 9. Tests

The feature should add focused tests before broader M3 commit-ordering and freelist tests.

### 9.1 Lifecycle Tests

- `beginRead()` returns a live read transaction and increments active reader accounting
- `beginWrite()` returns a live write transaction and marks the writer as active
- second writer begin attempt returns `WriteConflict`
- `close()` with an active reader returns `InvalidState`
- `close()` with an active writer returns `InvalidState`

### 9.2 State Machine Tests

- read transaction transitions from `active_read` to `closed_rollback`
- write transaction transitions from `active_write` to `closed_rollback`
- write transaction transitions from `active_write` to `committing` to `closed_commit`
- injected commit failure transitions `committing` to `closed_failed`
- terminal transactions reject later reads, writes, `commit()`, and helper use with `TransactionClosed`

### 9.3 Wrapper Tests

- `DB.get` and `DB.exists`, if retained, open and close a short read transaction
- `DB.put` and `DB.delete`, if retained, open a write transaction and call `rollback()` on pre-commit failure
- wrapper APIs do not leak active transaction accounting on success or failure

### 9.4 Error Tests

- mutating through a read transaction returns `ReadOnlyTransaction`
- `beginWrite()` on a read-only DB returns `ReadOnlyTransaction`
- exceeding `max_read_transactions` returns the documented lifecycle error consistently
- read-path corruption in a pinned snapshot returns `Corruption`
- injected storage write or sync failures during commit return `IoError` and leave the transaction terminal

## 10. Non-Goals

This feature does not include:

- crash recovery validation after restart
- metadata A/B recovery selection logic beyond what commit already needs
- full freelist pending-to-reusable promotion policy
- bucket or cursor APIs
- overflow-page support
- multi-process transaction locking
- performance tuning beyond basic lifecycle correctness

## 11. Acceptance Criteria

This feature is complete when:

- `beginRead()` and `beginWrite()` are the canonical M3 entry points for transaction lifecycle
- the implementation uses an explicit `TxState` enum with the required active, committing, closed, and failed states
- all legal lifecycle transitions are enforced and all illegal reuse paths fail predictably
- closed and failed transactions both reject later operations with `TransactionClosed`
- `commit()` success closes the writer and `commit()` failure leaves it terminal
- `rollback()` closes read and write transactions and releases DB accounting exactly once
- `close()` rejects active transactions with `InvalidState`
- any retained DB-level helpers are thin wrappers over explicit transactions and cannot bypass lifecycle or ownership rules
- public error behavior matches the M0 API sketch
- tests cover begin, commit, rollback, terminal reuse, writer exclusivity, wrapper cleanup, and close rejection

## 12. Follow-On Dependencies

Once this feature lands, later M3 work can build on it for:

- snapshot visibility tests against real tree reads
- dirty-page ownership and copy-on-write mutation plumbing
- commit ordering hooks needed by M4 crash-safety work
- freelist reader-safety promotion rules
