# Repository Guidelines

## Project Structure & Module Organization
This repository is currently design-first. Core planning lives in [README.md](/D:/代码D/zi6db/README.md), [ROADMAP.md](/D:/代码D/zi6db/ROADMAP.md), and [MILESTONES.md](/D:/代码D/zi6db/MILESTONES.md). Detailed technical specs are under `docs/design/m0/`, including storage format, recovery rules, API sketches, and crash/corruption test plans. GitHub collaboration assets live in `.github/`, including issue templates, the PR template, and contributor-facing agent notes.

When implementation begins, keep production Zig code under a dedicated `src/` tree and tests under `test/` or module-adjacent `*_test.zig` files. Match any new directories to the milestone or subsystem they implement.

## Build, Test, and Development Commands
There is no runnable database yet, so there is no enforced build pipeline in this repository today. For documentation work, use:

- `git status` to inspect pending changes before committing.
- `rg "keyword" docs` to find related design decisions quickly.
- `git log --oneline` to review recent commit patterns.

If Zig code is added, document the exact local commands in the same PR, for example `zig build`, `zig test test/all.zig`, or subsystem-specific test entry points.

## Coding Style & Naming Conventions
Write Markdown with short sections, explicit headings, and stable terminology. Prefer lowercase kebab-case file names such as `storage-format.md` and `crash-safety-matrix.md`. Keep design documents concrete: define invariants, failure modes, and versioning rules instead of vague intent.

For future Zig code, follow standard Zig formatting via `zig fmt`, use descriptive lower_snake_case identifiers where appropriate, and keep module names aligned with subsystem names.

## Testing Guidelines
Testing is currently specified rather than implemented. Use `docs/design/m0/crash-and-corruption-test-spec.md` as the baseline for future validation work. When adding executable tests, name them after the behavior under test, cover persistence and crash-safety paths, and link each new test area back to the relevant milestone or design document.

## Commit & Pull Request Guidelines
Git history and `.github/agents/commit-and-pr-specialist.agent.md` indicate Conventional Commit style: `docs(m0): add crash recovery matrix`, `feat(storage): ...`, `fix(wal): ...`. Keep commits focused and scoped.

Branch names should also be conventional and reviewable: use `feat/<feature>`, `fix/<feature>`, `docs/<feature>`, `chore/<feature>`, or another standard change type followed by a short kebab-case description. Examples: `feat/page-cache`, `chore/github-templates`. Do not use personal or tool-generated prefixes such as `codex/...`.

Pull requests should follow `.github/pull_request_template.md` exactly: include a summary, linked issue (`Closes #123`), change type, concrete testing steps, and any relevant screenshots or terminal output. Update docs whenever behavior, format rules, or milestone assumptions change.
