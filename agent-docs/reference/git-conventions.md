# Git Conventions

## PR titles

Use Conventional Commits:

`<type>[optional scope][!]: <summary>`

- `type` is required.
- `scope` is optional.
- `!` marks a breaking change.
- `.github/workflows/pr-title.yml` enforces the title on `pull_request` updates.

Allowed types:
- `feat`
- `fix`
- `chore`
- `test`
- `bench`
- `perf`
- `refactor`
- `docs`
- `ci`
- `revert`
- `deps`

## Scopes

Scopes are optional. Use a short stable area name when it helps routing, e.g. `engine`, `rpc`, `storage`, `provider`, `trie`, `stages`, `cli`, `bench`, `deps`, `release`.

## Commit / merge subject shape

Recent history follows the same Conventional Commit style for merged subjects, often with a PR suffix such as `(#23376)`.
Keep summaries short and imperative.

## PR expectations

Before review, expect CI to check at least:
- formatting (`cargo fmt --all --check`)
- clippy / lint workflows
- docs build and generated CLI docs consistency
- unit, integration, and e2e test workflows

Relevant workflow files: `.github/workflows/lint.yml`, `.github/workflows/unit.yml`, `.github/workflows/integration.yml`, `.github/workflows/e2e.yml`.

A label workflow exists at `.github/workflows/label-pr.yml`.

## Changelog fragments

The `.changelog/` fragment workflow was removed.
Do not add changelog fragment files unless maintainers explicitly ask for them.

## Release tags

`.github/workflows/release.yml` triggers on tags matching `v*`.
Observed formats include stable tags like `v1.10.2` and rc tags like `v1.10.0-rc.1`.

The release workflow strips the leading `v` and verifies the tag matches the Cargo version prefix. This allows rc tags to match the same base crate version.

Tags containing `-rc` are published as prereleases.
