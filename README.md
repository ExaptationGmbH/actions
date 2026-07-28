# actions

Shared CI/CD building blocks. Reusable GitHub Actions live in their own
subdirectory and are **versioned independently** via release-please.

## Actions

| Action | Purpose |
|--------|---------|
| [`build-harbor-image`](build-harbor-image/) | Set up Buildx, log in to Harbor, and build + push a Docker image with GHA layer caching. Replaces the hand-rolled setup-buildx + login + build-push block in every container repo. |

## Consuming an action

Reference it by path + a component-scoped tag:

```yaml
uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v0
```

Pin however tightly you like:

- `@build-harbor-image-v0` — sliding major; tracks all `0.x` releases. Pre-1.0 this can move across breaking changes.
- `@build-harbor-image-v0.0` — sliding minor; safest pin before 1.0.0 (recommended for now).
- `@build-harbor-image-v0.0.1` — immutable exact release.
- `@<sha>` — a specific commit.

> The repo must be **public** (or the consumer must live in the same org with
> private-action access enabled) for cross-repo `uses:` to resolve.

## Releasing (release-please monorepo)

`.github/workflows/release-please.yml` watches `main`. On conventional-commit
pushes it opens a **separate release PR per action** (`separate-pull-requests`),
each with its own version and `CHANGELOG.md`. Merging an action's release PR:

1. cuts the immutable tag + GitHub Release, e.g. `build-harbor-image-v0.0.1`
   (`include-component-in-tag`), and
2. moves the sliding `build-harbor-image-v0` / `-v0.0` tags to that commit (the
   `*-major-tag` job in the workflow).

So `build-harbor-image` can ship `1.4.0` while a future action stays at `0.2.0`
— versions never collide because every tag and changelog is component-scoped.

### Adding a new action

1. Create `<action-name>/action.yml` (+ `README.md`).
2. Add a package entry under `packages` in `release-please-config.json`
   (`release-type: simple`, `component`/`package-name` = the dir name).
3. Add `"<action-name>": "0.0.0"` to `.release-please-manifest.json`.
4. Add a matching per-component `outputs` block and a sliding-tag job to
   `release-please.yml` (copy the `harbor-major-tag` job, swap the names).

Config source of truth: [`release-please-config.json`](release-please-config.json)
· [`.release-please-manifest.json`](.release-please-manifest.json).

> release-please needs the repo setting **Settings → Actions → General → Allow
> GitHub Actions to create and approve pull requests** enabled.
