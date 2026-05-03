# cDashboard Agent Instructions

## Scope

These instructions apply to the whole repository.

## Branch Workflow (Required)

- Never commit directly to `main`.
- Create a working branch before making changes.
- Keep each branch focused on one logical change.
- Use short kebab-case branch names with a type prefix:
  - `feat/<topic>`
  - `fix/<topic>`
  - `refactor/<topic>`
  - `docs/<topic>`
  - `test/<topic>`
  - `ci/<topic>`
  - `chore/<topic>`

## Commit Style (Required)

Use Conventional Commits for all commits.

Format:

`<type>(<scope>): <short summary>`

Allowed types:

- `feat`
- `fix`
- `perf`
- `refactor`
- `docs`
- `test`
- `build`
- `ci`
- `chore`
- `revert`

Breaking changes must use one of the following:

- `type(scope)!: summary`
- a `BREAKING CHANGE:` footer in the commit body

Examples:

- `feat(poller): add per-service timeout handling`
- `fix(tui): keep last known rows on API failure`
- `docs(spec): clarify stale data behavior`

## SemVer Rules (Required)

Use Semantic Versioning tags: `vMAJOR.MINOR.PATCH`.

Determine the next release bump from commits since the last version tag:

- `major`: any commit marked as breaking (`!` or `BREAKING CHANGE:`)
- `minor`: any `feat` commit (when no major bump exists)
- `patch`: `fix`, `perf`, `revert`, and user-facing `refactor` (when no major/minor bump exists)
- `none`: `docs`, `test`, `chore`, and `ci` only

No release should be created when the change set is only `docs`/`test`/`chore`/`ci`.

No initial release tag should be created until real application code is shipped.

## Changelog Rules (Required)

Maintain `CHANGELOG.md` in Keep a Changelog style.

- Keep an `Unreleased` section at the top.
- Add dated release sections only when a version is actually released.
- Group entries under:
  - `Added`
  - `Changed`
  - `Fixed`
  - `Removed`
  - `Security`
- Include only user-facing or operationally meaningful changes.
- Do not add noise-only entries for internal `docs`/`test`/`chore`/`ci` work.

## Release Procedure

When a release is requested:

1. Read commits since the last tag and determine the required bump.
2. Update `CHANGELOG.md` from `Unreleased` into a new version section.
3. Create a release commit using: `chore(release): vX.Y.Z`.
4. Create tag: `vX.Y.Z`.
5. Push commit and tag.
