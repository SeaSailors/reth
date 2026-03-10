# Git Conventions

This repository follows a Conventional Commits-style format for PR titles and (in practice) merge commit subjects.

## Commit / PR Title Format

### Required shape

Use the following format (Conventional Commits):

```text
<type>[optional scope][!]: <short summary>
```

- `type` is required.
- `scope` is optional and appears as `type(scope): ...`.
- `!` indicates a breaking change (e.g. `feat!: ...`, `chore(node)!: ...`).

A GitHub workflow validates PR titles against this format and an allowlist of types (see `.github/workflows/pr-title.yml`).

### Allowed types

The PR title linter allows these types:

- `feat` — new feature
- `fix` — bug fix
- `chore` — maintenance / housekeeping
- `test` — tests
- `bench` — benchmarks
- `perf` — performance improvements
- `refactor` — refactoring without behavior change (ideally)
- `docs` — documentation
- `ci` — CI/CD changes
- `revert` — reverts
- `deps` — dependency updates

### Scopes

Scopes are optional but commonly used and should be a short, stable identifier for the impacted area.

Observed scopes in recent history include:

- `engine`
- `flashblocks`
- `stages`
- `download`
- `execution-types`
- `storage-api`
- `storage`
- `db`
- `consensus`
- `provider`
- `trie`
- `chain-state`
- `cli`
- `bench`
- `deps`
- `release`

Prefer a scope when it helps reviewers immediately route the change.

### Examples (from `git log`)

Real examples from this repo:

- `fix: handle incomplete receipts gracefully in receipt root task (#21285)`
- `fix(engine): clear execution cache when block validation fails (#21282)`
- `feat(download): resumable snapshot downloads with auto-retry (#21161)`
- `perf(storage): batch trie updates across blocks in save_blocks (#21142)`
- `refactor(stages): reuse history index cache buffers in collect_history_indices (#21017)`
- `chore(deps): weekly cargo update (#21167)`
- `docs: document minimal storage mode in pruning FAQ (#21025)`
- `ci: update to tempoxyz (#21176)`
- `revert: undo Chain crate, add LazyTrieData to trie-common (#21155)`

Notes:

- Many merged commit subjects include a PR number suffix like `(#21285)`. Include it if your workflow does so automatically; otherwise it is optional.
- Keep summaries imperative and concise (what changed and why), avoid trailing punctuation.

## PR Expectations (from CI configuration)

### Title

- PR title must be Conventional Commit formatted and use an allowed `type`.
- Breaking changes should be marked with `!` in the title.

### CI gates

Pull requests and merge queue entries run extensive checks (see `.github/workflows/lint.yml`, `.github/workflows/unit.yml`, and others). Expect to satisfy, at minimum:

- Formatting: `cargo fmt --all --check`
- Lints: `cargo clippy ... -D warnings` (multiple configurations)
- Docs build and generated docs checks (including a `git diff --exit-code` check for generated CLI docs)
- Workspace checks (including feature propagation / dependency checks)

### Merge workflow

- CI is configured to run on `pull_request`, and also on `merge_group` for merge queue runs.
- This implies the project likely uses GitHub’s merge queue / merge groups for `main`.

### Auto-labeling

- A workflow applies labels to newly opened PRs (`.github/workflows/label-pr.yml`).

No PR template was found under `.github/` (e.g., `PULL_REQUEST_TEMPLATE.md`), so follow existing PR norms: clear title, clear description, and include testing notes.

## Release Tagging / Versioning (discoverable rules)

### Tag format

- Releases are triggered by pushing tags matching `v*` (see `.github/workflows/release.yml`).
- Observed tags use SemVer-like versions:

  - Stable: `v1.10.2`, `v1.10.1`, `v1.10.0`, ...
  - Release candidates: `v1.10.0-rc.2`, `v1.10.0-rc.1`, ...

### Cargo version must match tag

The release workflow verifies that the crate version in Cargo metadata matches the pushed tag prefix:

- It strips the leading `v` from the tag.
- It checks that `Cargo.toml` version starts with the tag value.
- This allows `vX.Y.Z-rc.N` tags to match a `Cargo.toml` version starting with `X.Y.Z`.

### Pre-releases

- Tags containing `-rc` are treated as pre-releases when drafting the GitHub Release.

## Suggested Contributor Workflow (small)

1. Create a feature branch from `main`.
2. Make focused commits (small, reviewable).
3. Open a PR with a Conventional Commit title (`type(scope): ...`), and add `!` if it is breaking.
4. Ensure local checks pass before requesting review (at least `cargo fmt`, `cargo clippy`, relevant tests).
5. Address review feedback; keep the PR title compliant (the linter runs on edits/synchronization).
