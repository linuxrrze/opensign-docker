# opensign-server

Docker image for the [OpenSign](https://github.com/OpenSignLabs/OpenSign)
**backend API** (Node/Express + the Parse.js layer + Mailgun/SMTP/DoSpace
integrations + LibreOffice DOCX->PDF conversion), built via GitHub Actions and
published to `ghcr.io/<owner>/opensign-server` -- see
[`.github/workflows/opensign-server-rebuild.yml`](../.github/workflows/opensign-server-rebuild.yml).

## How it's built

There's a **local Dockerfile** at
[`docker/Dockerfile`](docker/Dockerfile) that mirrors upstream's
`apps/OpenSignServer/Dockerhubfile` (same `node:22.14.0` base, same
`apt-get install -y libreoffice`, same `npm install` + `node index.js`). CI
clones the OpenSign monorepo fresh on every run, checks out the exact release
tag (tracked in `.opensign-server-version`), and builds the **local**
Dockerfile with the clone root as the build context -- so the Dockerfile's
`COPY apps/OpenSignServer/...` lines resolve against the upstream source. The
build installs **LibreOffice** (headless DOCX -> PDF conversion,
`apt-get install -y libreoffice`) and runs `npm install`. The server listens
on port 8080 and mounts the Parse.js API at `$PARSE_MOUNT` (`/app` by default,
i.e. the API is reachable under `/app/...`).

The one deliberate deviation from upstream's Dockerfile is the
`apt-get update && apt-get -y dist-upgrade` layer (see the comment in
[`docker/Dockerfile`](docker/Dockerfile)). `node:22.14.0` is a Debian
(bookworm) image whose baked-in OS packages -- including everything
LibreOffice pulls in -- carry fixable CRITICAL/HIGH CVEs; this refreshes them
so the workflow's Trivy gate passes. The workflow also tracks the base
image's registry digest (`.opensign-server-base-digest`) and rebuilds when
Debian re-publishes `node:22.14.0` with security fixes, so the OS layer stays
current without needing an OpenSign release.

## Files

| File | Used by CI? | Purpose |
| --- | --- | --- |
| `docker/Dockerfile` | yes | Local Dockerfile (mirrors upstream's `apps/OpenSignServer/Dockerhubfile` + a `dist-upgrade` layer); built with the monorepo clone as the build context. |
| `.trivyignore` | yes | Deliberately-accepted fixable CRITICAL/HIGH CVEs the `dist-upgrade` can't remove (the `node:22.14.0` bundled-npm tree + OpenSign's pinned `lodash` / `marked` / `minimist` / `underscore` app deps -- the 3 CRITICALs). Passed to Trivy via `--ignorefile`. |
| `.opensign-server-version` | yes | Last upstream release tag (e.g. `v2.41.3`) CI built; compared against the GitHub releases API to decide whether to rebuild. Updated automatically by CI. |
| `.opensign-server-base-digest` | yes | Registry digest of the `node:22.14.0` base image the last build used; CI rebuilds when it moves (Debian security updates reaching the node image) so the Dockerfile's `dist-upgrade` keeps the OS layer current. |

## Version pinning

Upstream publishes real semver release tags (`vX.Y.Z`). CI checks out the exact
tagged release rather than floating `main` -- more reproducible, and it keeps
the server in lockstep with the `opensign-client` (both track the same tag).
