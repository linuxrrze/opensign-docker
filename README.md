# opensign-docker

Docker images for a self-hosted [OpenSign](https://github.com/OpenSignLabs/OpenSign)
stack, built and published by GitHub Actions to the GitHub Container Registry
at `ghcr.io/<owner>/<image>`.

| Directory | What | Workflow |
| --- | --- | --- |
| [`opensign-client/`](opensign-client/README.md) | Web client image (React/Vite SPA) | [`opensign-client-rebuild.yml`](.github/workflows/opensign-client-rebuild.yml) |
| [`opensign-server/`](opensign-server/README.md) | Backend API image (Node/Express + Parse.js + LibreOffice) | [`opensign-server-rebuild.yml`](.github/workflows/opensign-server-rebuild.yml) |
| [`opensign/`](opensign/README.md) | Deployment hub: `docker-compose.yml.sample` + `.env.sample` for the whole stack | -- |

Each workflow checks the latest upstream OpenSign release daily (and the
registry digest of the `node:22.14.0` base image), and on a change -- or on a
push touching the image's Dockerfile / `.trivyignore` / workflow, or a manual
run -- clones the release, builds, adds a startup banner, gates on a Trivy
scan, pushes `:<version>`, `:<version>-<YYYYMMDD>` and `:latest`, and commits
the new version/digest back to `main`. Details and conventions:
[`.github/workflows/README.md`](.github/workflows/README.md).

## Repository setup on GitHub

No secrets to create: both workflows authenticate with the built-in
`GITHUB_TOKEN` (the workflow files request `contents: write` +
`packages: write`).

1. **Actions permissions.** *Settings -> Actions -> General*: allow Actions
   to run. The workflows declare their own `permissions:`, so the default
   "Read repository contents" token setting is fine -- unless an
   organization policy caps the token at read-only, in which case allow
   read/write there.
2. **Branch protection.** The "Record built version" step pushes straight to
   `main` as `github-actions[bot]`. If `main` is protected, allow GitHub
   Actions to bypass the rule (or drop the protection) -- otherwise the image
   is published but the version bump is rejected, and the next scheduled
   run rebuilds the same release again.
3. **Package visibility.** The first push creates the `opensign-client` /
   `opensign-server` packages, linked to this repo via the
   `org.opencontainers.image.source` label. New ghcr.io packages are
   **private** by default: make them public under the package's *Package
   settings* if hosts should pull without logging in; otherwise run
   `docker login ghcr.io` on the host with a PAT that has `read:packages`.
4. **Verify.** *Actions -> \<workflow\> -> Run workflow*. The manual run
   always builds; check that build, scan and push succeed and that a bot
   commit updates `<image>/.<image>-version` / `-base-digest` if they
   moved.

### Scheduled runs

GitHub disables `schedule:` triggers in a **public** repo after 60 days
without repository activity. The bot's version-bump commits only happen when
something changed, so a quiet upstream can let the schedule lapse -- GitHub
emails a warning first; re-enable it from the workflow's page in the
Actions tab. Cron times are UTC.

## Deploying

See [`opensign/README.md`](opensign/README.md): copy the two `.sample` files,
set `IMAGE_PREFIX=ghcr.io/<owner>` (lowercase) and the app settings in `.env`,
`docker compose up -d`.
