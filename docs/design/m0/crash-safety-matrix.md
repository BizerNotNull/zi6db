# M0 Crash Safety Matrix

| Crash / failure point | What may be on disk | Could `commit()` already have returned success? | Recovery result |
| --- | --- | --- | --- |
| Before any non-metadata page write | old committed state only | No | select old metadata |
| During non-metadata page writes | mixture of old state plus unreachable new pages | No | select old metadata |
| After non-metadata page writes, before pre-metadata sync | old state plus maybe complete new data pages | No | select old metadata |
| After pre-metadata sync, before metadata write | old state plus durable new data pages | No | select old metadata |
| During metadata write | old metadata, or torn/invalid new metadata, or fully written new metadata | No | select newest valid metadata; if neither valid, `Corruption` |
| After metadata write, before metadata sync | new metadata may be readable and valid | No | old or new metadata may be selected depending on validation |
| After metadata sync succeeds | new metadata and referenced pages durable within failure model | Yes | select new metadata |
| Metadata sync returns error / cannot confirm success | new metadata may or may not already be durable | No | reopening and recovery determine whether old or new metadata wins |
| File growth fails before metadata write | partial tail extension may exist | No | select old metadata |
| Header write/create path fails during initial database creation | incomplete bootstrap file may exist | No | fail open/create; incomplete file is invalid until recreated |

## Notes

- Unreachable pages left behind by a failed commit are not corruption by themselves.
- Metadata is the only commit switch point.
- Equal-generation metadata pages are always treated as `Corruption` in M0.
