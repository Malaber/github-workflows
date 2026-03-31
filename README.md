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
