# CLAUDE.md

Guidance for Claude Code (or any AI assistant) working in this repository.

## What this repo is

Docker images for a self-hosted OpenSign stack, built and published by
GitHub Actions to `ghcr.io/<owner>/<image>`. See
[`.github/workflows/README.md`](.github/workflows/README.md) for the
workflow conventions.

## Structure

- `opensign-client/`, `opensign-server/`: one directory + one
  `.github/workflows/<name>-rebuild.yml` per image. Dockerfile under
  `<name>/docker/`, built with a fresh upstream monorepo clone as context.
- `opensign/`: deployment hub (`docker-compose.yml.sample`, `.env.sample`);
  references the images by their pushed tag (`${IMAGE_PREFIX}/<name>:latest`),
  never a local build context.
- `.github/scripts/`: shared helpers (base digest, startup banner, Trivy).

## Compose files and secrets

- The real `docker-compose.yml` and `opensign/.env` are gitignored -- never
  commit them. Edit the `.sample` files instead.

## Workflow conventions

- Never truncate a pipe early (`head`, or `awk` with an early `exit`) under
  `set -o pipefail`.
- The "Record built version" step ends with a retry-with-rebase loop, not a
  bare `git push`.
- Secrets go through step-level `env:`, never inline into a bash string.
- Every workflow sets `run-name:` directly under `name:`.
- Every image gets the startup banner step right after "Build image"
  (`.github/scripts/add-banner.sh`), same tags. Never install figlet into an
  image.
- ghcr.io names must be lowercase: derive the image name from
  `GITHUB_REPOSITORY_OWNER` lowercased, never hardcode the owner.

## Verifying a new or changed pipeline

Force a version mismatch (decrement `<name>/.<name>-version`), run the
workflow, and confirm it rebuilds, pushes, and bumps the file back on its own.

## Git workflow

- `git fetch origin` and check for new bot commits before every commit -- CI
  pushes version-bump commits on its own.
- For multi-line commit messages, write to a temp file and use
  `git commit -F <file>`.
