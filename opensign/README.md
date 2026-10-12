# opensign

Self-hosted [OpenSign](https://github.com/OpenSignLabs/OpenSign) -- an open
source DocuSign alternative (document signing, templates, verification emails,
multi-party signing). Deployed here as a small stack; the two app containers
are built and published by this repo's CI, the database is stock Mongo.

| Container | Image | Built by |
| --- | --- | --- |
| `client` | `${IMAGE_PREFIX}/opensign-client:latest` | [`opensign-client/`](../opensign-client/) (CI) |
| `server` | `${IMAGE_PREFIX}/opensign-server:latest` | [`opensign-server/`](../opensign-server/) (CI) |
| `mongo` | `mongo:8.2` (stock, no auth) | pulled |

## How the app is built

OpenSign is a monorepo whose client/server `Dockerfile`s live upstream at
`apps/OpenSign/Dockerhubfile` / `apps/OpenSignServer/Dockerhubfile`. The two CI
workflows **clone the monorepo fresh, check out the exact release tag** (tracked
in `opensign-client/.opensign-client-version` and
`opensign-server/.opensign-server-version`), and build this repo's own
Dockerfiles (`opensign-client/docker/Dockerfile`,
`opensign-server/docker/Dockerfile` -- upstream's plus an `apt-get
dist-upgrade` layer so the Trivy gate passes) with the clone root as the build
context. The client builds a Vite SPA (`node:22.14.0` base, port 3000,
served by the app's own `server.cjs`); the server is a Node/Express + Parse.js
API (`node:22.14.0` base, port 8080, with LibreOffice installed for
headless DOCX -> PDF conversion).

## Deploy

1. `cp docker-compose.yml.sample docker-compose.yml && cp .env.sample .env`
2. Edit `.env`:
   - `IMAGE_PREFIX` -- `ghcr.io/<owner>` of the GitHub repo whose CI publishes
     the images (lowercase). If the packages are private, `docker login ghcr.io`
     first with a PAT that has `read:packages`.
   - `OPEN_PUBLIC_HOST` -- the DNS name the app is reachable at (both the web
     client and the `/api` prefix; traefik routes on it).
   - `MASTER_KEY` -- change from the upstream default (12 random chars).
   - `MONGODB_URI` -- leave as `mongodb://mongo:27017/OpenSignDB` unless you
     point mongo elsewhere.
   - Email: set `SMTP_ENABLE=true` + the `SMTP_*` values (or the `MAILGUN_*`
     values) so verification + signing emails actually send.
   - Optional: S3-compatible storage (`USE_LOCAL=false` + `DO_*`) and a signing
     certificate (`PFX_BASE64` + `PASS_PHRASE`).
3. `docker compose up -d`

The first browser visit runs OpenSign's own setup (create the first user /
admin, pick the app language). No migration step is needed here -- OpenSign's
Parse.js schema manages itself against Mongo.

## Routing (traefik)

Upstream ships a `Caddyfile` that reverse-proxies `/api/*` to the server with
the `/api` prefix stripped. That's replicated here with traefik labels instead
of a Caddyfile:

- `opensign-web` router: `Host(OPEN_PUBLIC_HOST) && !PathPrefix(/api)` -> client:3000
- `opensign-api` router: `Host(OPEN_PUBLIC_HOST) && PathPrefix(/api)` -> server:8080,
  through an `opensign-api-strip` middleware (`StripPrefix(/api)`).

The strip mirrors upstream's caddy: the server mounts its Parse.js API at
`/app` (`PARSE_MOUNT`) and serves uploaded documents under `/files/...`, so a
browser request to `https://HOST/api/files/x` is stripped to `server:/files/x`.
`REACT_APP_SERVERURL` (client) and `SERVER_URL`/`PUBLIC_URL` (server) are
derived from `OPEN_PUBLIC_HOST` in the compose file, so they always point at
the `/api` route.

> Note: a real deployment needs a reverse-proxy network for traefik to reach
> these containers -- add it per deployment. The sample leaves that out, like
> the upstream sample.

## Files

| File | Used by CI? | Purpose |
| --- | --- | --- |
| `docker-compose.yml.sample` | no | The full stack. Copy to `docker-compose.yml` and fill in `OPEN_PUBLIC_HOST`. |
| `.env.sample` | no | Template for `.env` -- app id, MASTER_KEY, MONGODB_URI, storage + email + signing config. |
| `opensign-client/` | yes | The web-client image (CI builds it); see that dir's README. |
| `opensign-server/` | yes | The Node/Parse server image (CI builds it); see that dir's README. |

## MongoDB version

`mongo` is pinned to `8.2` because existing data was written by 8.2 (its
`featureCompatibilityVersion`, FCV, is `8.2`). mongod cannot start on data
whose FCV is newer than itself -- an older image fails with
`Invalid feature compatibility version value '8.2'; expected ...` -- so the
pin must never go below the data's FCV. Check it with:

```sh
docker exec opensign-mongo mongosh --quiet --eval \
  'db.adminCommand({getParameter: 1, featureCompatibilityVersion: 1})'
```

To upgrade later: bump the image one release at a time, start it, then
raise the FCV to the new version (`db.adminCommand({setFeatureCompatibilityVersion: "<x.y>", confirm: true})`).

## Security

- **Mongo has no auth.** OpenSign ships Mongo without credentials. The compose
  sample maps `27018->27017` for local `mongosh` access only -- keep that (and
  27017) off the public internet, as upstream explicitly warns.
- **`MASTER_KEY`.** Change it from the upstream default; it's the secret that
  grants full access to the Parse data layer.
- **Email.** The app warns if no email provider is configured, and
  account-verification + signing-notification flows won't work until one is.
- **Signing certificate.** A self-signed cert (OpenSign's docs include the steps)
  won't show a valid green tick in Adobe Acrobat -- use one from an AATL-approved
  CA if that matters.
