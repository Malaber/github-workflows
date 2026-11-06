# github-workflows

Reusable GitHub Actions workflows for Malaber repositories.

## Included workflows

### `.github/actions/stamp-version`

Stamps a version into repository files using declarative regex replacements.

- accepts a `version`
- accepts a JSON `replacements` array
- each replacement defines `path`, `pattern`, and `replacement`
- `replacement` may contain `{version}`

### `reusable-version.yml`

Computes the next release version from git tags.

- pushes to `main` get the next stable patch version
- other branches get `-rc.<run>` prerelease versions
- outputs `base_version`, `release_version`, `git_tag`, and `prerelease`

### `reusable-ansible-collection-release.yml`

Builds and publishes an Ansible collection release from an exact source commit.

- checks out the caller repository at the provided `source_sha`
- runs the shared `stamp-version` action with caller-provided replacement specs
- builds the collection tarball
- creates the git tag when needed
- publishes the tarball to the matching GitHub Release

### `reusable-dependabot-sync.yml`

Keeps application repositories aligned with the canonical files under `templates/`.

- compares the caller's Dependabot configuration and sync workflow with the canonical templates
- renders the caller with the immutable commit SHA currently on this repository's `main` branch
- updates only the `automation/sync-dependabot` branch
- opens or refreshes a pull request instead of pushing to the default branch
- reuses an existing synchronization pull request when one is already open

Call it from a small scheduled workflow in each application repository:

```yaml
name: Sync Dependabot configuration

on:
  schedule:
    - cron: "23 5 * * 1"
  workflow_dispatch:

jobs:
  sync:
    permissions:
      contents: write
      pull-requests: write
    uses: Malaber/github-workflows/.github/workflows/reusable-dependabot-sync.yml@<commit-sha>
```

## Example

```yaml
jobs:
  version:
    uses: Malaber/github-workflows/.github/workflows/reusable-version.yml@main
    with:
      seed_version: "0.1.0"

  release-collection:
    needs:
      - version
    uses: Malaber/github-workflows/.github/workflows/reusable-ansible-collection-release.yml@main
    permissions:
      contents: write
    secrets: inherit
    with:
      source_sha: ${{ github.sha }}
      release_version: ${{ needs.version.outputs.release_version }}
      git_tag: ${{ needs.version.outputs.git_tag }}
      prerelease: ${{ needs.version.outputs.prerelease }}
      stamp_replacements: >-
        [{"path":"pyproject.toml","pattern":"^version = \"[^\"]+\"$","replacement":"version = \"{version}\""}]
      artifact_name: my-namespace-my-collection-${{ needs.version.outputs.release_version }}
```

## App Store creative assets

`reusable-appstore-creative.yml` renders caller-owned SVG artwork around a current
screenshot artifact. It validates fully decoded opaque 5244×2950 RGB PNGs and
requires editable SVGs for both header and search-results assets. No Apple
credentials, asset replacement, release/tag mutation, or AI image generation.

```yaml
creative:
  needs: screenshots
  uses: Malaber/github-workflows/.github/workflows/reusable-appstore-creative.yml@<reviewed-commit-sha>
  with:
    screenshot_artifact: appstore-iphone
    screenshot_glob: en-US/02-cooking-companion*.png
    renderer: marketing/creative/render.py
    requirements: marketing/requirements.txt
    output_artifact: appstore-creative
```

The caller's renderer must accept `--screenshot PATH --output DIRECTORY` and emit
`header.svg`, `header.png`, `search-results.svg`, `search-results.png`. Keep branding,
licensed fonts and editable source artwork in the caller repository. Outline text
and embed image references for portable exported SVGs. Inkscape is installed on
Ubuntu 24.04; pin its package/container version in a fork when byte-identical
rendering across future runner updates is required. Consumers should pin this
workflow to an immutable reviewed SHA. Download the output artifact in the
existing release-packaging job. Require human visual approval in App Store
Connect before publishing new artwork.

Creative jobs accept `job_timeout_minutes` (default 30). Dependency installation
has a separate 15-minute bound, uses Ubuntu's canonical package mirror and omits
recommended packages. Transport failures fail the job without automatic retries.
The remaining budget covers rendering and uploading; a slow install cannot consume
the entire former 15-minute job budget.
