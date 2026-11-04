# AGENTS.md — pod-crabbox-systemd

The **systemd-init lease image** for the Crabbox `containerd` provider: the
`pod-crabbox-systemd` image (`charly.yml`) and the `systemd-init` candy
(`candy/systemd-init/`) it composes on top of `opencharly/pod-crabbox`. This repo
owns exactly the systemd delta — the init entrypoint, the enabled units, the
masked hvc0 getty, the pid-max drop-in, the krun shutdown hook and the runner's
.NET dependencies. The lease contract itself belongs to
`opencharly/pod-crabbox`; a change to it lands there, not here.

Canonical files:

- `charly.yml` — the image: `base: crabbox.pod-crabbox` at a pinned tag,
  `entrypoint: [/sbin/init]`, `user: root` (systemd refuses a non-root PID 1).
- `candy/systemd-init/charly.yml` — the variant's packages, plan steps and
  `check:` steps.
- `.github/workflows/image.yml` — the image build; it triggers on
  `tag-on-merge` **completion** (`workflow_run`), never on `push: tags`
  (`opencharly/.github#277`: tag-on-merge mints tags through the REST API, which
  emits no push event).
- `.github/workflows/tag-on-merge.yml` — the org's CalVer tag + `CHANGELOG/`
  dispatcher, installed by the org bootstrap.
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-image:image` — the `candy:` image entry (base, user, entrypoint).
- `/charly-image:layer` — the candy body: `distro:` packages, plan step verbs,
  and the `check:` surface.
- `/charly-distros:cachyos` — the CachyOS base and Arch package semantics.
- `/charly-check:check` — the check verbs, the ADE requirement, and the R10 gate.
- `/charly-internals:git-workflow` — before any git/PR action.

## What is unique here

- **The image tag IS the git tag.** The box is named after the repo, and
  `.github/workflows/image.yml` passes the newest `v*` tag (minus its `v`) to
  `charly box build --tag`, which uses a supplied tag verbatim. Do not add a
  clock-derived tag.
- **The base is imported, never copied.** `import: crabbox:
  '@github.com/opencharly/pod-crabbox:<tag>'` — advancing that pin is the only
  way a base-contract fix reaches this image. Run `charly box reconcile` for a
  cross-repo pin alignment.
- **systemd is PID 1 through the OCI entrypoint, not `init:`.** The
  `systemd-container` init lands separately (`opencharly/charly#1030`); until it
  is in the engine pin this image expresses the same thing with
  `entrypoint: [/sbin/init]` plus explicit `multi-user.target.wants` symlinks.
  When `systemd-container` is in the pinned engine, the image moves onto it in
  one cutover (entrypoint + enabled units + mask move together).
- **Enabling a unit is a `link:`, not a `service:` entry.** Declaring a
  `service:` on a container image selects supervisord and injects it; this image
  is systemd-as-PID-1 and must not compose supervisord.
- **Never `seccomp=unconfined`, never `--no-sandbox`.** The browser variant ships
  the narrow seccomp profile; this base does not weaken the boundary.

## Build / validate / test

- `charly box validate` at the repo root — the structural check on the image and
  its candy (including all imported bases at their pinned tags).
- `charly box generate pod-crabbox-systemd` — inspect the emitted Containerfile
  and the baked `ai.opencharly.description` plan.
- `charly check box pod-crabbox-systemd` — the build-scope `check:` steps.
- The PR image build (`.github/workflows/image.yml`, `pull_request` path) builds
  the image and publishes nothing; the merge-time `workflow_run` path publishes
  `ghcr.io/opencharly/pod-crabbox-systemd:<git-tag>`.

## Modify this repo

- Keep the pin of `pod-crabbox` and `distro-cachyos` in step; a base fix must land
  in its owning repo first, then this repo's pin advances.
- A package may be added only from a named source (the base candy, an existing
  `layer-*`/`pod-*` repo's Arch section, or the provider's docs). Record the
  source in a comment, as `icu`/`krb5`/`libunwind`/`lttng-ust` do.
- `zlib` must NOT be added: CachyOS ships `zlib-ng-compat`, which provides libz —
  an explicit `zlib` is an unresolvable conflict (measured in
  `opencharly/pod-github-runner`).
- Every claim in the plan needs a deterministic `check:`. A package that merely
  installs without a probe is not proof.

## Landing

- PR-only. Every change lands through a pull request; the org-required
  `charly/pr-validator` validates the diff and body and arms native auto-merge on
  PASS. Direct pushes to `main` are blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR
  body IS the changelog.
- The authoritative rulebook is the umbrella `AGENTS.md` in
  `opencharly/opencharly`. Do not restate its rules here.
