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
`buildkit-native-export-v0.1.0`. Publishing the builder with `pack_ref` set to such a
tag produces a builder tag that matches the pinned lifecycle + pack tag exactly.

### Fork branch layout (matches the pack + lifecycle forks)

- **`fork-main`** — the fork's DEFAULT (and protected) branch. Carries the fork-only
  tooling: `publish-builder.yml` (manual), the `test-builder.yml` smoke-test disable,
  this `.kiro` steering, and the `builder.toml` lifecycle pins. You dispatch
  `publish-builder.yml` from here (dispatchable workflows must live on the default
  branch).
- **`buildkit-native-export`** — the pristine, code/config-only branch: `main` plus ONLY
  the two `builder.toml` `[lifecycle].uri` changes, as a single commit. No fork CI, no
  `.kiro`; the smoke test is enabled (upstream-clean).
- **`buildkit-native-export-with-history-and-kiro`** — full dev history + `.kiro` +
  workflows; the safety copy and where development happens.
- **`main`** — the repo baseline. This repo is standalone (no `upstream` remote), so
  `main` is `origin`'s own baseline, not a mirror of some external upstream.

See `builder-fork-branch-restructure-runbook.md` (in dot-kiro-files) for how this layout
is produced and maintained.

## The custom POC workflow: `publish-builder.yml`

`.github/workflows/publish-builder.yml` is the fork-specific workflow that builds
and pushes the multi-arch builder image. It is separate from the upstream
`push-image.yml` (which triggers on GitHub releases and uses `scripts/publish.sh`).

Triggers:
- **`workflow_dispatch` only** (manual run). The old `push`-to-branch and
  `push`-a-tag auto-publish triggers were removed so the builder is published
  on-demand only, consistent with `publish-lifecycle.yml` and `publish-pack.yml`.

Input:
- **`pack_ref`** — the `jericop/pack` branch or tag to build the pack binary from
  (default `buildkit-native-export`). The published builder image tag is DERIVED from
  this ref (a branch publishes a moving tag named after it; a
  `buildkit-native-export-v*` ref publishes that immutable tag verbatim).

What it does:
1. **Resolve builder tag** — derives the builder tag from `pack_ref` (verbatim); falls
   back to the `DEFAULT_BUILDER_TAG` env (`buildkit-native-export`) only if `pack_ref`
   is somehow empty.
2. **Builds `pack` from source** — checks out `jericop/pack` at `pack_ref` and
   `go build`s the `pack` binary (Go version taken from pack's `go.mod`). The fork's
   `pack` is required because it understands the BuildKit multi-arch flags.
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

- `jericop/ubuntu-noble-builder:buildkit-native-export` — current builder published
  with `pack_ref=buildkit-native-export`, bundling the `buildkit-native-export-v0.1.0`
  lifecycle. Multi-arch manifest (amd64 + arm64).
- `jericop/ubuntu-noble-builder:buildkit-native-export-v0.1.0` — immutable
  semver-tagged builder published from the matching git tag; bundles the pinned
  lifecycle + is created with the matching pack tag.
- `jericop/ubuntu-noble-builder:skip-chown-poc` — original POC builder bundling the
  `skip-chown-poc` lifecycle (`-skip-chown` only). Kept distinct so it is not
  overwritten; used by `pr-compliance-app` CI.

Keep distinct tags per lifecycle so updating one does not break consumers of the
other.

## How to publish manually

1. Ensure `builders/builder/builder.toml`'s `[lifecycle].uri` points at the intended
   lifecycle tag (this decides which `jericop/lifecycle` the builder bundles).
2. Dispatch the workflow from the default branch (`fork-main`), passing the pack ref to
   build the pack binary from:
   ```bash
   gh workflow run publish-builder.yml --repo jericop/ubuntu-noble-builder \
     --ref fork-main \
     -f pack_ref=buildkit-native-export
   ```
   Or: Actions -> "publish-builder" -> "Run workflow". The builder image tag is derived
   from `pack_ref`. See `publish-images-runbook.md` in dot-kiro-files for the full
   cross-repo publish flow (lifecycle -> builder ordering, etc.).

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
