# semantic-release

Reusable common-repo template for automated releases. Consumer contract: [README.md](README.md).

## Layout

- `.common-repo.yaml` — `self:` maintains this repo; the outer pipeline exports `src/**` into consumer roots and inherits conventional-commits.
- `src/cog.toml` — release defaults; consumers add artifact-building `pre_bump_hooks` here through TOML merging.
- `src/.github/workflows/release.yaml` — CI → local bump/hooks → push tag → publish release and `dist/*` assets → update major tag → push bump commit. Preserve publication order.
- `src/.github/workflows/ci.yaml` — reusable fallback CI; `if-exists: preserve` protects consumers' existing workflow.
- Root `.github/`, `cog.toml`, `.pre-commit-config.yaml` — this template's own maintenance configuration.
- Keep agent indexes outside `src/`: every payload file is exported to consumers.

## Tools and validation

- **Cocogitto** (`cog`) — version planning, bump hooks, tags, changelog; **GitHub CLI** (`gh`) — release publication.
- **prek** — repository hooks; **conventional-pre-commit** — local commit-message validation.
- Run `prek install` on new checkouts/worktrees; validate with `prek run --all-files`, `common-repo validate`, and `common-repo apply --dry-run`.
- `cog check --from-latest-tag` checks commit history. CI also runs an executable `script/test` when a consumer supplies one.
- Source and `self:` pipelines have separate variable scopes; keep local GitHub App defaults in both. Consumer `with: template-vars` overrides source defaults; see README for credentials and branch protection.

## Maintaining this index

- Update the affected `AGENTS.md` files in the same change when paths, responsibilities, commands, dependencies, or conventions change.
- Keep indexes brief: record semantic entry points and non-obvious constraints; link to existing documentation instead of duplicating it.
- Add a directory index only when it provides useful navigation beyond its parent; omit generated, vendored, and fixture trees.
- Every `AGENTS.md` must have a sibling `CLAUDE.md` containing only `@AGENTS.md`.
