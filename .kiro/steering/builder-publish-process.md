---
inclusion: always
---

# Builder Image Publish Process (jericop/ubuntu-noble-builder)

This fork publishes an experimental Cloud Native Buildpacks **builder image** used
by the BuildKit multi-arch work in `jericop/pack`. The builder bundles a patched
lifecycle so the lifecycle can run inside BuildKit.

The BuildKit work has been consolidated onto a **single builder-agnostic `buildkit`
backend** (build-then-finalize). The old OCI-layout export mode and its
`-pull-run-image` lifecycle flag were removed, so the builder no longer bundles a
lifecycle that carries `-pull-run-image`.

## Branch and tag naming

All three repos share one branch name for this work: **`buildkit-native-export`**
(`jericop/pack`, `jericop/cnb-lifecycle`, `jericop/ubuntu-noble-builder`).

Feature-line semver tags are also shared verbatim across the repos, e.g.
`buildkit-native-export-v0.1.0`. Publishing the builder from a git tag produces a
builder tag that matches the pinned lifecycle + pack tag exactly.

## The custom POC workflow: `publish-builder.yml`

`.github/workflows/publish-builder.yml` is the fork-specific workflow that builds
and pushes the multi-arch builder image. It is separate from the upstream
`push-image.yml` (which triggers on GitHub releases and uses `scripts/publish.sh`).

Triggers:
- `push` to the `buildkit-native-export` branch,
- `push` of a `buildkit-native-export-v*` git tag (publishes the builder under the
  tag name verbatim), and
- `workflow_dispatch` (manual run).

What it does:
1. **Resolve builder tag** — on a git-tag push the builder tag is the tag name
   verbatim (`refs/tags/<tag>` -> `<tag>`); otherwise it falls back to the
   `DEFAULT_BUILDER_TAG` env (`buildkit-native-export`).
2. **Builds `pack` from source** — checks out `jericop/pack` at the ref in the
   `PACK_REF` env (`buildkit-native-export`) and `go build`s the `pack` binary
   (Go version taken from pack's `go.mod`). The fork's `pack` is required because
   it understands the BuildKit multi-arch flags.
3. **`publish` job (matrix per arch)** — on `ubuntu-24.04` (amd64) and
   `ubuntu-24.04-arm` (arm64) runners, runs:
   ```
   pack builder create <DOCKERHUB_ORG>/ubuntu-noble-builder:<tag>-<arch> \
     --config ./builders/builder/builder.toml \
     --target linux/<arch> \
     --publish
   ```
   Each arch is built natively on its own runner (no QEMU) and pushed as a
   per-arch tag (`...-amd64` / `...-arm64`). Platforms are limited to
   `linux/amd64` + `linux/arm64`.
4. **`manifest` job** — after both arch images publish, creates and pushes a
   Docker manifest list tag that points at the two per-arch images.

## What the builder bundles (builder.toml)

`builders/builder/builder.toml` defines the build image, buildpacks, order, run
image, and — critically — the **lifecycle**:

```toml
[lifecycle]
  uri = "docker://docker.io/jericop/lifecycle:buildkit-native-export-v0.1.0"
```

This pins the builder-agnostic buildkit lifecycle line: single `buildkit` backend,
the `io.buildpacks.lifecycle.prepared-metadata` label, and `-skip-chown`, with no
OCI-layout mode / `-pull-run-image`. It is an immutable semver tag published from
the `jericop/cnb-lifecycle` `buildkit-native-export-v0.1.0` git tag.

The lifecycle images are published from the `jericop/cnb-lifecycle` repo (see its
`.github/workflows/publish-lifecycle.yml` and its buildkit steering).

## Image tags

- `jericop/ubuntu-noble-builder:buildkit-native-export` — current builder from a
  branch push, bundling the `buildkit-native-export-v0.1.0` lifecycle. Multi-arch
  manifest (amd64 + arm64).
- `jericop/ubuntu-noble-builder:buildkit-native-export-v0.1.0` — immutable
  semver-tagged builder published from the matching git tag; bundles the pinned
  lifecycle + is created with the matching pack tag.
- `jericop/ubuntu-noble-builder:skip-chown-poc` — original POC builder bundling the
  `skip-chown-poc` lifecycle (`-skip-chown` only). Kept distinct so it is not
  overwritten; used by `pr-compliance-app` CI.

Keep distinct tags per lifecycle so updating one does not break consumers of the
other.

## How to publish manually

1. Ensure `builders/builder/builder.toml`'s `[lifecycle].uri` points at the
   intended lifecycle tag and the tags/refs in `publish-builder.yml`
   (`DEFAULT_BUILDER_TAG`, `PACK_REF`) are what you want.
2. Actions -> "publish-builder" -> "Run workflow" (or push to
   `buildkit-native-export`, or push a `buildkit-native-export-v*` tag).
   CLI: `gh workflow run publish-builder.yml --repo jericop/ubuntu-noble-builder`.

## Configuration (repo secrets)

- `PAKETO_BUILDPACKS_DOCKERHUB_USERNAME` / `PAKETO_BUILDPACKS_DOCKERHUB_PASSWORD`
  — Docker Hub credentials used by `azure/docker-login`.
- `DOCKERHUB_ORG` — the Docker Hub org/namespace to publish under (e.g. `jericop`).

## Relationship to other repos

- `jericop/cnb-lifecycle` (`buildkit-native-export`) — publishes the
  `jericop/lifecycle:*` images this builder bundles.
- `jericop/pack` (`buildkit-native-export`) — built from source here and the
  consumer of the published builder image for multi-arch buildkit builds.
- `jericop/pr-compliance-app` — CI that exercises builds against the builder image.
