# `build-harbor-image`

Composite GitHub Action that bundles the Docker-image-to-Harbor build into a
single step:

```
setup-qemu (optional) → setup-buildx → login (Harbor) → build-push (+ GHA cache)
```

Instead of hand-rolling the same four steps in every container repo — with
slightly different credential names and tag wiring — call this action and pass
your registry, credentials, image, and tags.

There is **no default registry** — `registry` is required. Wire it to a repo
variable (e.g. define `HARBOR_REGISTRY` under Settings → Secrets and variables →
Actions → Variables and pass `${{ vars.HARBOR_REGISTRY }}`), or hardcode the
hostname.

## Usage

The caller still runs `actions/checkout` (the build context is your tree) and
resolves its own version string.

```yaml
- uses: actions/checkout@v4

- uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v0
  with:
    registry: ${{ vars.HARBOR_REGISTRY }}          # required — e.g. harbor.example.com
    username: ${{ secrets.HARBOR_USERNAME }}
    password: ${{ secrets.HARBOR_PASSWORD }}
    image: myproject/myimage
    version: ${{ steps.version.outputs.VERSION }}   # e.g. 1.2.3 → pushes :1.2.3 and :latest
```

That single step logs into your Harbor registry, builds `./Dockerfile`
for `linux/amd64` with GHA layer caching, and pushes
`<registry>/myproject/myimage:1.2.3` and `:latest`.

## Inputs

| Input | Required | Default | Notes |
|-------|----------|---------|-------|
| `registry` | **yes** | — | Harbor hostname, e.g. `harbor.example.com`. No default — pass it explicitly (e.g. `${{ vars.HARBOR_REGISTRY }}`). |
| `username` | **yes** | — | Harbor robot/user. Pass a secret or var. |
| `password` | **yes** | — | Harbor token. Pass a secret. |
| `image` | **yes** | — | Repo path **without** registry, e.g. `myproject/myimage`. |
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

### Single image + version/latest

```yaml
- uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v0
  with:
    registry: ${{ vars.HARBOR_REGISTRY }}
    username: ${{ secrets.HARBOR_USERNAME }}
    password: ${{ secrets.HARBOR_PASSWORD }}
    image: myproject/myimage
    version: ${{ steps.version.outputs.VERSION }}
    build-args: VERSION=${{ steps.version.outputs.VERSION }}
```

### Several images in one repo, parallel matrix + per-image cache

```yaml
strategy:
  matrix:
    app: [api, worker, web]
steps:
  - uses: actions/checkout@v4
    with: { ref: ${{ steps.ref.outputs.tag }} }
  - uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v0
    with:
      registry: ${{ vars.HARBOR_REGISTRY }}
      username: ${{ secrets.HARBOR_USERNAME }}
      password: ${{ secrets.HARBOR_PASSWORD }}
      image: myproject/${{ matrix.app }}
      tags: ${{ steps.ref.outputs.tag }}          # keep the v-prefix if a downstream tool needs it
      cache-scope: ${{ matrix.app }}
      build-args: |
        APP=${{ matrix.app }}
        VERSION=${{ steps.ref.outputs.tag }}
```

### Multi-arch

```yaml
- uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v0
  with:
    registry: ${{ vars.HARBOR_REGISTRY }}
    username: ${{ secrets.HARBOR_USERNAME }}
    password: ${{ secrets.HARBOR_PASSWORD }}
    image: myproject/myimage
    version: ${{ steps.release.outputs.tag_name }}
    platforms: linux/amd64,linux/arm64
    qemu: true
```

### Build-only on PRs, push on main + metadata-action tags

```yaml
- uses: docker/metadata-action@v5
  id: meta
  with:
    images: ${{ vars.HARBOR_REGISTRY }}/myproject/myimage
    tags: |
      type=sha,prefix=
      type=raw,value=latest,enable={{is_default_branch}}
      type=raw,value=main-{{date 'YYYYMMDDHHmmss'}}-{{sha}},enable={{is_default_branch}}

- uses: ExaptationGmbH/actions/build-harbor-image@build-harbor-image-v0
  with:
    registry: ${{ vars.HARBOR_REGISTRY }}
    username: ${{ secrets.HARBOR_USERNAME }}
    password: ${{ secrets.HARBOR_PASSWORD }}
    image: myproject/myimage
    tags: ${{ steps.meta.outputs.tags }}          # full refs passed through
    push: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
```

## Multiple registries

This action targets a single Harbor registry. To also mirror an image to another
registry (e.g. Docker Hub) in one build, add a second `docker/login-action` step
for that registry and pass both registries' fully-qualified refs via `tags` — the
action passes full refs through untouched.

## Migration

Replace the `setup-buildx` + `login` + `build-push` block in a repo's workflow
with one `uses:` step. Keep your existing checkout, version-resolution, and
trigger config. Credential input names are uniform (`username` / `password`)
regardless of what the repo calls its secrets — just wire the repo's secret to
the input.
