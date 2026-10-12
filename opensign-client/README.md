# opensign-client

Docker image for the [OpenSign](https://github.com/OpenSignLabs/OpenSign)
**web client** (the React/Vite SPA + its tiny static file server), built via
GitHub Actions and published to `ghcr.io/<owner>/opensign-client` -- see
[`.github/workflows/opensign-client-rebuild.yml`](../.github/workflows/opensign-client-rebuild.yml).

## How it's built

There's a **local Dockerfile** at
[`docker/Dockerfile`](docker/Dockerfile) that mirrors upstream's
`apps/OpenSign/Dockerhubfile` (same `node:22.14.0` base, same
`npm install` + `vite build` + `server.cjs` static server). CI clones the
OpenSign monorepo fresh on every run, checks out the exact release tag
(tracked in `.opensign-client-version`), and builds the **local** Dockerfile
with the clone root as the build context -- so the Dockerfile's
`COPY apps/OpenSign/...` lines resolve against the upstream source. The
published base is `node:22.14.0`; the build runs `npm install` +
`npm run build` (Vite) and serves the `build/` output on port 3000 via the
app's own `server.cjs` static server (entrypoint
`apps/OpenSign/entrypoint.sh` injects `REACT_APP_SERVERURL` into a runtime
`env.js` at container start).

The one deliberate deviation from upstream's Dockerfile is the
`apt-get update && apt-get -y dist-upgrade` layer (see the comment in
[`docker/Dockerfile`](docker/Dockerfile)). `node:22.14.0` is a Debian
(bookworm) image whose baked-in OS packages carry fixable CRITICAL/HIGH
CVEs; this refreshes them so the workflow's Trivy gate passes. The workflow
also tracks the base image's registry digest
(`.opensign-client-base-digest`) and rebuilds when Debian re-publishes
`node:22.14.0` with security fixes, so the OS layer stays current without
needing an OpenSign release.

## Files

| File | Used by CI? | Purpose |
| --- | --- | --- |
| `docker/Dockerfile` | yes | Local Dockerfile (mirrors upstream's `apps/OpenSign/Dockerhubfile` + a `dist-upgrade` layer); built with the monorepo clone as the build context. |
| `.trivyignore` | yes | Deliberately-accepted fixable CRITICAL/HIGH CVEs the `dist-upgrade` can't remove (the `node:22.14.0` bundled-npm tree + a couple of OpenSign app deps). Passed to Trivy via `--ignorefile`. |
| `.opensign-client-version` | yes | Last upstream release tag (e.g. `v2.41.3`) CI built; compared against the GitHub releases API to decide whether to rebuild. Updated automatically by CI. |
| `.opensign-client-base-digest` | yes | Registry digest of the `node:22.14.0` base image the last build used; CI rebuilds when it moves (Debian security updates reaching the node image) so the Dockerfile's `dist-upgrade` keeps the OS layer current. |

## Version pinning

Upstream publishes real semver release tags (`vX.Y.Z`). CI checks out the exact
tagged release rather than floating `main` -- more reproducible, and it matches
the tag the app's own `npm run version` step stamps into the build at compile
time.
