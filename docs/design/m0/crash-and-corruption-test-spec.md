# M0 Crash And Corruption Test Spec

This document turns the M0 design freeze into a future fault-injection and validation test plan.

## 1. Header Tests

- reject wrong header magic
- reject unsupported format major/minor
- reject invalid endianness marker
- reject page sizes below `4096` or above `65536`
- reject header checksum mismatch
- reject header/metadata duplicated-field conflicts

## 2. Metadata Selection Tests

- only metadata A valid -> select A
- only metadata B valid -> select B
- A newer than B -> select A
- B newer than A -> select B
- both invalid -> `Corruption`
- equal generation with different contents -> `Corruption`
- equal generation with identical contents -> `Corruption`
- newer generation with decreasing `txn_id` -> `Corruption`

## 3. Critical Page Validation Tests

- selected `root_page_id` out of range -> `Corruption`
- selected `freelist_page_id` out of range -> `Corruption`
- selected root page has wrong `page_type` -> `Corruption`
- selected freelist page has wrong `page_type` -> `Corruption`
- selected root page checksum mismatch -> `Corruption`
- selected freelist page checksum mismatch -> `Corruption`

## 4. Commit Crash Matrix Tests

- crash before non-metadata writes -> old state survives
- crash during non-metadata writes -> old state survives
- crash after non-metadata sync but before metadata write -> old state survives
- crash during metadata write with torn page -> old state survives if new metadata invalid
- crash after metadata write but before metadata sync -> old or new state may win
- crash after metadata sync -> new state must win

## 5. I/O Failure Injection Tests

- `ENOSPC` during tail growth before metadata write -> `commit()` fails, old state remains authoritative
- short write of non-metadata page -> `commit()` fails
- short write of metadata page -> `commit()` fails, recovery chooses only valid metadata
- pre-metadata sync failure -> `commit()` fails
- metadata sync failure -> `commit()` fails and outcome remains ambiguous until reopen
- file create flush failure in `strict` mode -> create/open path fails

## 6. Freelist Tests

- deleted pages enter `pending_free[current_txn_id]`
- pages in `pending_free` are not reused while an older reader is active
- after older readers close, pending pages become reusable
- restart in single-process mode can migrate eligible pending groups to reusable
- unreachable tail pages are not auto-added to freelist during recovery

## 7. Compatibility Tests

- unsupported major version -> `IncompatibleVersion`
- unsupported required feature flag -> `IncompatibleVersion`
- supported file with corrupted metadata -> `Corruption`
- supported file with corrupted header -> `Corruption`

## 8. API Lifecycle Tests

- second writer is rejected with `WriteConflict`
- write attempt in `read_only` database -> `ReadOnlyTransaction`
- reuse of committed transaction -> `TransactionClosed`
- reuse of rolled-back transaction -> `TransactionClosed`
- `close()` with active transactions -> `InvalidState`

## 9. Verification Expectations

Every future crash-safety test should assert:

- what bytes may legally remain on disk
- whether `commit()` could have returned success before the injected fault
- which metadata slot recovery should select
- whether the expected result is successful open or `Corruption`
