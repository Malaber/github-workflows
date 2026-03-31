# AGENTS.md

## Scope

These instructions apply to the entire repository.

## Repository purpose

This repository stores reusable GitHub Actions workflows that are consumed from
other Malaber repositories.

The current workflows cover two responsibilities:

- compute a release version from existing git tags
- build and publish an Ansible collection release from an exact source commit
- stamp versions into caller repositories through a shared composite action

## Design summary

### `reusable-version.yml`

This workflow is responsible only for release version calculation.

- it checks out the caller repository with full tag history
- it derives the next stable patch release from `v*` tags
- `main` receives a stable version
- non-`main` refs receive `-rc.<run_number>` prerelease versions
- it returns `base_version`, `release_version`, `git_tag`, and `prerelease`

### `reusable-ansible-collection-release.yml`

This workflow is responsible only for publishing the Ansible collection once the
caller has already decided which source commit and version to release.

- it checks out the caller repository at `source_sha`
- it runs the shared stamp action with caller-provided replacement specs
- it builds the collection artifact
- it uploads the artifact between jobs
- it creates the git tag if it does not exist yet
- it publishes the artifact to the matching GitHub Release

### `.github/actions/stamp-version`

This composite action performs regex-based file updates inside the checked-out
caller repository.

- it accepts a `version`
- it accepts JSON replacement specs
- each spec declares `path`, `pattern`, and `replacement`
- `replacement` may contain `{version}`
- each pattern must match exactly once

## Caller responsibilities

The caller repository remains responsible for repository-specific policy.

- decide when releases are allowed to run
- prepare or commit version-stamped source on the default branch
- pass replacement specs that describe the repo's versioned files
- choose the artifact naming convention
- grant `contents: write` and use `secrets: inherit` when invoking the release workflow

## Change guidelines

- Keep workflows generic and reusable across repositories.
- Prefer inputs over repo-specific assumptions.
- Preserve the strict split between version calculation and release publishing.
- Update `README.md` when workflow inputs, outputs, or usage expectations change.
- Validate edited workflow YAML before committing.

## Validation

At minimum, validate changed workflow files with a YAML parser before pushing.
