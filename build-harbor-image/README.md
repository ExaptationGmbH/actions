# `build-harbor-image`

Composite GitHub Action that bundles the Docker-image-to-Harbor build used
across the Exaptation repos into a single step:

```
setup-qemu (optional) → setup-buildx → login (Harbor) → build-push (+ GHA cache)
```

Every repo that ships a container — slackodoro, sophie-rae, scentral-park,
konsens, plato, plato-www, hub, mycorra — was hand-rolling the same four steps
with slightly different credential names and tag wiring. This action is the
single source of truth for that pattern.

## Usage

The caller still runs `actions/checkout` (the build context is your tree) and
resolves its own version string.

```yaml
- uses: actions/checkout@v4

- uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v1
  with:
    username: ${{ secrets.HARBOR_USERNAME }}
    password: ${{ secrets.HARBOR_PASSWORD }}
    image: konsens/konsens
    version: ${{ steps.version.outputs.VERSION }}   # e.g. 1.31.1 → pushes :1.31.1 and :latest
```

That single step logs into `harbor.exaptation.company`, builds `./Dockerfile`
for `linux/amd64` with GHA layer caching, and pushes
`harbor.exaptation.company/konsens/konsens:1.31.1` and `:latest`.

## Inputs

| Input | Required | Default | Notes |
|-------|----------|---------|-------|
| `registry` | no | `harbor.exaptation.company` | Harbor hostname. |
| `username` | **yes** | — | Harbor robot/user. Pass a secret or var. |
| `password` | **yes** | — | Harbor token. Pass a secret. |
| `image` | **yes** | — | Repo path **without** registry, e.g. `plato/hub`. |
| `tags` | no | `""` | Newline/comma list. Bare suffixes (`1.2.3`) are expanded to full refs; entries containing `/` are passed through unchanged (feed `docker/metadata-action` output here). Wins over `version`. |
| `version` | no | `""` | Convenience: pushes `<version>` (+ `latest`) when `tags` is empty. Strip leading `v` yourself if unwanted. |
| `latest` | no | `true` | Also push `:latest` when deriving from `version`. |
| `context` | no | `.` | Build context. |
| `file` | no | `Dockerfile` | Dockerfile path. |
| `platforms` | no | `linux/amd64` | Set `linux/amd64,linux/arm64` for multi-arch (also set `qemu: true`). |
| `build-args` | no | `""` | Newline-separated build args. |
| `target` | no | `""` | Dockerfile stage to target. |
| `push` | no | `true` | Set `false` to build-only (PR verification). Login is skipped when false. |
| `cache-scope` | no | `""` | Per-image GHA cache scope; set a distinct value per image when a repo builds several in parallel. |
| `qemu` | no | `false` | Set up QEMU for cross-arch builds. |

## Outputs

| Output | Description |
|--------|-------------|
| `image-ref` | Registry-qualified path (`registry/image`, no tag). |
| `tags` | Fully-resolved newline list that was pushed. |
| `digest` | Digest of the pushed image. |
| `metadata` | `build-push-action` result metadata JSON. |

## Examples

### Single image + version/latest (slackodoro, konsens, plato-www)

```yaml
- uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v1
  with:
    username: ${{ secrets.HARBOR_NAME }}
    password: ${{ secrets.HARBOR_KEY }}
    image: konsens/konsens
    version: ${{ steps.version.outputs.VERSION }}
    build-args: VERSION=${{ steps.version.outputs.VERSION }}
```

### Several images in one repo, parallel matrix + per-image cache (scentral-park)

```yaml
strategy:
  matrix:
    app: [admin, fragrance-index, mcp, crawler, enricher]
steps:
  - uses: actions/checkout@v4
    with: { ref: ${{ steps.ref.outputs.tag }} }
  - uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v1
    with:
      username: ${{ secrets.HARBOR_USERNAME }}
      password: ${{ secrets.HARBOR_PASSWORD }}
      image: scentral-park/${{ matrix.app }}
      tags: ${{ steps.ref.outputs.tag }}          # keep the v-prefix for Flux semver
      cache-scope: ${{ matrix.app }}
      build-args: |
        APP=${{ matrix.app }}
        VERSION=${{ steps.ref.outputs.tag }}
```

### Multi-arch (mycorra)

```yaml
- uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v1
  with:
    username: ${{ secrets.HARBOR_USERNAME }}
    password: ${{ secrets.HARBOR_PASSWORD }}
    image: ${{ vars.HARBOR_PROJECT || 'library' }}/mycorra
    version: ${{ steps.release.outputs.tag_name }}
    platforms: linux/amd64,linux/arm64
    qemu: true
```

### Build-only on PRs, push on main (hub) + metadata-action tags

```yaml
- uses: docker/metadata-action@v5
  id: meta
  with:
    images: ${{ secrets.REGISTRY_URL }}/plato/hub
    tags: |
      type=sha,prefix=
      type=raw,value=latest,enable={{is_default_branch}}
      type=raw,value=main-{{date 'YYYYMMDDHHmmss'}}-{{sha}},enable={{is_default_branch}}

- uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v1
  with:
    registry: ${{ secrets.REGISTRY_URL }}
    username: ${{ secrets.REGISTRY_USERNAME }}
    password: ${{ secrets.REGISTRY_PASSWORD }}
    image: plato/hub
    tags: ${{ steps.meta.outputs.tags }}          # full refs passed through
    push: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
```

## Notes on the outlier (`plato`)

`plato/.github/workflows/docker-publish.yml` mirrors the **same** image to
Docker Hub **and** Harbor in one `build-push-action` call. This action targets a
single registry (Harbor). Either keep plato's dual-push workflow as-is, or add a
second `docker/login-action` for Docker Hub and pass both registries' refs via
`tags` — the action already passes full refs through untouched.

## Migration

Replace the `setup-buildx` + `login` + `build-push` block in each repo's
workflow with one `uses:` step. Keep your existing checkout, version-resolution,
and trigger config. Credential input names are now uniform (`username` /
`password`) regardless of what each repo calls its secrets — just wire the
repo's secret to the input.
