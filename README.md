# semantic-release

A [common-repo](https://github.com/common-repo/common-repo) source template for conventional commits and semantic releases.

## What's Included

Files distributed from `src/`:

| File | Purpose |
|---|---|
| `.releaserc.yaml` | semantic-release config (conventionalcommits preset, changelog, git, GitHub release, major-tag) |
| `commitlint.config.js` | commitlint config extending `@commitlint/config-conventional` |
| `.github/workflows/commitlint.yml` | PR commit lint workflow using `wagoid/commitlint-github-action` |
| `.github/workflows/release.yaml` | Semantic release workflow (triggers on push to main, uses GitHub App token) |

## Usage

> **Requires common-repo ≥ 0.28.4** for source-declared filtering and template variable overrides.

### Add to an existing `.common-repo.yaml`

```yaml
# .common-repo.yaml
- repo:
    url: https://github.com/common-repo/semantic-release
    ref: v0.2.0
```

That's it. The source repo's `.common-repo.yaml` handles scoping to `src/` and stripping the prefix.

### Apply

```bash
cr diff    # preview changes
cr apply   # apply
```

## Distributed Files

After applying, your repo will receive:

```
.github/workflows/
├── commitlint.yml     ← PR commit lint (wagoid/commitlint-github-action)
└── release.yaml       ← semantic release on push to main (GitHub App token)
.releaserc.yaml        ← semantic-release config (conventionalcommits, changelog, GitHub release, major-tag)
commitlint.config.js   ← commitlint config (@commitlint/config-conventional)
```

`release.yaml` is a template — its GitHub App credentials are substituted from `template-vars` at apply time.

## Template Variables

The release workflow is a template with three variables:

| Variable | Purpose |
|---|---|
| `GH_APP_ID_VAR` | GitHub Actions `vars.*` name for the App ID |
| `GH_APP_KEY_SECRET` | GitHub Actions `secrets.*` name for the App private key |
| `GH_APP_OWNER` | GitHub App installation owner (org or user) |

These are required — consumers must provide all three via `template-vars`. They are used by `actions/create-github-app-token` to generate a token with write permissions for creating releases and pushing tags/changelogs.

### Example

```yaml
- repo:
    url: https://github.com/common-repo/semantic-release
    ref: v2.0.0
    with:
      - template-vars:
          GH_APP_ID_VAR: MY_APP_ID
          GH_APP_KEY_SECRET: MY_APP_KEY
          GH_APP_OWNER: my-org
```

This renders the workflow with `${{ vars.MY_APP_ID }}`, `${{ secrets.MY_APP_KEY }}`, and `owner: my-org`.

## Customization

### Release workflow

The included `release.yaml` contains only the semantic-release job. It does **not** include a publish step (goreleaser, Docker, etc.) — add that in your repo's own workflow or extend via common-repo merge operators.

The workflow calls `./.github/workflows/ci.yaml` as a prerequisite. Your repo must have a CI workflow at that path (or remove the `ci` job dependency).

### Excluding files

If you only want a subset:

```yaml
- repo:
    url: https://github.com/common-repo/semantic-release
    ref: v1.1.0
    with:
      - include: ["src/.releaserc.yaml", "src/commitlint.config.js"]
      - rename:
          - "^src/(.*)$": "$1"
```

### Overriding `.releaserc.yaml`

Use common-repo's YAML merge operator to patch specific fields:

```yaml
- repo:
    url: https://github.com/common-repo/semantic-release
    ref: v1.1.0
- yaml:
    source: my-releaserc-overrides.yaml
    dest: .releaserc.yaml
    path: plugins
    array_mode: replace
```
