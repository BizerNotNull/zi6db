# M0 Decision Freeze Table

This table records the default choices that later milestones must obey.

| Area | Decision | Frozen default | May later milestones change it? |
| --- | --- | --- | --- |
| Page size | default/min/max | `4096 / 4096 / 65536` | No, except via format-version upgrade |
| Header bootstrap | fixed prefix size | `512` bytes | No |
| Endianness | byte order | little-endian | No |
| Header mutability | dynamic state in header | forbidden | No |
| Metadata ordering | primary selector | `generation` | No |
| Tx ordering | auxiliary selector | `txn_id` monotonic with generation | No |
| Metadata slots | switch rule | always write inactive A/B slot | No |
| Initial metadata | bootstrap state | only A valid, B invalid | No |
| Checksum algorithm | page/header checksum | `CRC32C` | No |
| Checksum coverage | normal pages | includes page id, type, generation, flags, body | No |
| Header corruption | recovery policy | return `Corruption` | No |
| Header/meta conflict | conflict policy | return `Corruption` | No |
| I/O model | storage path | positional read/write | No without re-review |
| Commit success point | API success | only after metadata sync succeeds | No |
| Commit failure ambiguity | API policy | recovery is authoritative after ambiguous failure | No |
| Freelist persistence | pending state | persist `pending_free` on disk | No |
| Reuse rule | reader safety | pending pages reusable only after older readers exit | No |
| Durability modes | labels | `strict`, `normal`, `unsafe` reserved | Existing meanings may not weaken |
| Strict durability | sync policy | two sync boundaries | No |
| Locking floor | early support | single-process multi-reader, single writer | Can extend, not weaken |
| Cross-process write safety | pre-M7 handling | may reject outright | Yes, can extend in M7 |
| Overflow format | page type reservation | reserved in M0 | No |
| Equal-generation metadata | startup behavior | return `Corruption` | No |
