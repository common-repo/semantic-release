# semantic-release

A [common-repo](https://github.com/common-repo/common-repo) source template
that distributes [cocogitto](https://docs.cocogitto.io/)-based semantic
release infrastructure: conventional-commits enforcement on PRs and a fully
automated release-on-merge workflow for `main`.

> The repo is named `semantic-release` for the *concept* (semver releases
> driven by conventional commits). The actual implementation is cocogitto,
> not the [`semantic-release`](https://github.com/semantic-release/semantic-release)
> JavaScript tool.

## What's Included

Files distributed from `src/`:

| File | Purpose |
|---|---|
| `cog.toml` | Cocogitto config (tag prefix `v`, ignore merge commits) |
| `.github/workflows/release.yaml` | Release-on-merge workflow (cocogitto bump + tag + GitHub Release) |

PR commit-lint is not shipped here — this template depends on
[`common-repo/conventional-commits`](https://github.com/common-repo/conventional-commits),
which installs `.github/workflows/conventional-commits.yaml` for that purpose.

## Usage

> Requires common-repo ≥ 0.28.4.

### Add to `.common-repo.yaml`

```yaml
- repo:
    url: https://github.com/common-repo/semantic-release
    ref: v0.3.0
    with:
      - template-vars:
          GH_APP_ID_SECRET: MY_APP_ID      # name of the `secrets.*` entry holding the GitHub App ID
          GH_APP_KEY_SECRET: MY_APP_KEY    # name of the `secrets.*` entry holding the private key
          GH_APP_OWNER: my-org             # GitHub App installation owner
```

Then apply:

```bash
cr diff    # preview
cr apply   # write
```

### Prerequisites in the consumer repo

- A **reusable CI workflow at `.github/workflows/ci.yaml`** (the release
  workflow calls it as a prerequisite).
- A **GitHub App** installed on the target repo with `contents: write`
  permissions. Expose its App ID and private key as `secrets.*` entries
  (values go in `template-vars` above).

## Template Variables

All three are required:

| Variable | Purpose |
|---|---|
| `GH_APP_ID_SECRET` | Name of the `secrets.*` entry holding the GitHub App's numeric ID |
| `GH_APP_KEY_SECRET` | Name of the `secrets.*` entry holding the App's private key PEM |
| `GH_APP_OWNER` | Owner (org or user) the App is installed on |

These are consumed by
[`actions/create-github-app-token`](https://github.com/actions/create-github-app-token)
to mint a token for pushing tags, creating releases, and committing bump
commits back to `main`.

---

## How releases work

On every push to `main`, the workflow:

1. Runs the consumer's CI.
2. Installs cocogitto and determines whether a new release is warranted
   (`cog bump --auto --dry-run`).
3. Runs any **pre-bump hooks** configured in the consumer's `cog.toml` —
   this is the extension point for producing artifacts that must exist at
   the tagged version *before* the bump commit lands on `main`.
4. Creates the bump commit and tag locally.
5. Pushes the tag, creates the GitHub Release (attaching anything in
   `./dist/`), updates the floating major-version tag (`vN`), and pushes the
   bump commit to `main` — in that order.

The strict ordering (**tag → release → assets → major tag → commit**) is
what makes this safe to combine with VCS-driven consumers. By the time the
bump commit hits `main`, the tag, the release, and any assets are already
in place — nothing reacting to the commit can lose a race.

## Publishing modes

Consumers plug into three well-defined slots:

| Slot | Where it runs | Owned by | Use when |
|---|---|---|---|
| **1. Pre-bump hook** | `cog bump` in the release workflow, before the commit+tag | Consumer (via `pre_bump_hooks` in `cog.toml`) | Artifact must exist at version `X.Y.Z` *before* the bump commit lands |
| **2. Bump + tag + GitHub Release** | Release workflow | This template | Always |
| **3. Post-release publish** | Separate workflow on `release: published` or `push: tags: v*` | Consumer | Artifact can be produced after the release exists |

### Mode reference

| Pattern | Example | Slot(s) |
|---|---|---|
| No artifact (Go lib, Action repo, common-repo template) | — | 2 |
| Artifact triggers deploy (image push → webhook → deploy) | Docker → GHCR → ArgoCD | 2 + 3 |
| Artifact attached to release | goreleaser binaries | 2 + 3 (or Slot 1 if goreleaser must own the release) |
| Artifact must exist before commit lands | Lambda zip consumed by Terraform Cloud | 1 + 2 |
| Artifact published to registry, commit references it | Container → ECR → TF ECS resource update | 1 + 2 |
| Version stamped into source | `-ldflags`, `__version__`, manifest files | 1 (stamp) + 2 |
| Pre-release channel | npm `next`, container `:edge` | 2 (with `prerelease: true`) + 3 |

---

## Slot 1 — Pre-bump hooks

Cocogitto's [`pre_bump_hooks`](https://docs.cocogitto.io/guide/bump.html#pre-bump-hooks)
run inside `cog bump`, before the commit and tag are created. The planned
version is available as `{{version}}`. Any files the hooks modify must be
`git add`'d by the hook to be included in the bump commit. A non-zero exit
aborts the whole release.

Hooks go in the consumer's `cog.toml`. The template ships a minimal
`cog.toml`; extend it by adding your own via a common-repo `toml:` merge:

```yaml
# .common-repo.yaml in the consumer repo
- repo:
    url: https://github.com/common-repo/semantic-release
    ref: v0.3.0
    with:
      - template-vars: { GH_APP_ID_VAR: ..., GH_APP_KEY_SECRET: ..., GH_APP_OWNER: ... }

- toml:
    source: cog-hooks.toml   # consumer-local file with hooks
    dest: cog.toml           # merge into the cog.toml shipped by the template
    array_mode: append
```

…with `cog-hooks.toml`:

```toml
pre_bump_hooks = [
  "scripts/my-prebuild.sh {{version}}",
]
```

### Convention: stage release assets in `./dist`

The release workflow uploads anything in `./dist/` to the GitHub Release.
Hooks that want their outputs attached to the release should write them
there.

### Example: Lambda zip consumed by Terraform

```toml
# cog-hooks.toml (merged into cog.toml)
pre_bump_hooks = [
  "scripts/build-lambda.sh {{version}}",   # produces ./dist/lambda.zip
]
```

Sequence at release time:

1. Hook builds `./dist/lambda.zip`.
2. `cog` creates the bump commit (with any `git add`'d files) and the
   `v{{version}}` tag — local only.
3. Workflow pushes the tag.
4. Workflow creates the GitHub Release and attaches `./dist/lambda.zip`.
5. Workflow pushes the bump commit to `main`.
6. Terraform Cloud picks up the commit and fetches the Lambda zip — it's
   already there.

### Example: container image consumed by Terraform via ECS

```toml
# cog-hooks.toml
pre_bump_hooks = [
  "docker build -t $ECR_REPO:{{version}} .",
  "docker push $ECR_REPO:{{version}}",
  "scripts/stamp-ecs-version.sh {{version}} && git add terraform/ecs.tf",
]
```

The hook pushes the image to ECR and stamps the new version into the
Terraform resource. The `git add` ensures the stamped file is part of the
bump commit. When the commit hits `main`, Terraform runs and finds the
image already in ECR.

---

## Slot 3 — Post-release publish

For anything that can run *after* the GitHub Release exists — most Docker
publishing, documentation deploys, npm/PyPI uploads, goreleaser, etc. —
add a separate workflow in the consumer repo:

```yaml
# .github/workflows/publish-image.yaml
name: Publish image

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ github.event.release.tag_name }}
      - uses: docker/build-push-action@v6
        with:
          tags: ghcr.io/${{ github.repository }}:${{ github.event.release.tag_name }}
          push: true
```

Multiple Slot-3 workflows can run in parallel on the same release (docs,
images, SDK publish, etc.) — each is independent.

---

## Customization

### Calling CI

The release workflow calls `./.github/workflows/ci.yaml` as a reusable
workflow (`workflow_call`). Consumers must either provide one at that path
or remove the `ci` job dependency via common-repo merge.

### Excluding files

If you only want a subset of what this repo ships:

```yaml
- repo:
    url: https://github.com/common-repo/semantic-release
    ref: v0.3.0
    with:
      - include: ["src/cog.toml"]
      - rename:
          - "^src/(.*)$": "$1"
```

### Overriding `cog.toml`

Use common-repo's TOML merge to extend the shipped config (e.g. to add
hooks, as shown above) or `exclude` it entirely and ship your own.

### Opting out of the GitHub Release step

Not currently supported via template-vars. If a consumer wants goreleaser
(or similar) to own the release, the cleanest path is to exclude
`.github/workflows/release.yaml` and ship a goreleaser-flavored variant.
Open an issue if you want this as a first-class toggle.

---

## Limitations and caveats

- **Single-package / single-artifact**. This template assumes one release
  version per push to `main`. Monorepos with per-package versioning should
  use a different template (cocogitto supports monorepo mode, but this
  workflow isn't wired for it).
- **Branch protection**. The release workflow pushes a bump commit to
  `main`. The GitHub App token must be allowed to bypass any required
  reviews / signed commits that would block a direct push.
- **Fast-forward required**. If `main` moves between the initial checkout
  and the final commit push, the push fails with non-fast-forward. The
  workflow does not auto-rebase. In practice the `concurrency: release`
  group plus the release job's short window make this rare.
- **GitHub App token scope**. The App needs `contents: write` on the
  target repo. If you publish to additional places (registries, other
  repos), grant those via the workflow's own secrets or additional tokens.
