# OpenSign web client (React/Vite SPA + its static file server).
#
# Mirrors upstream's apps/OpenSign/Dockerhubfile -- same node:22.14.0 base,
# same npm install + vite build + server.cjs static server -- but the build
# runs from a fresh clone of the monorepo (the workflow checks out the tracked
# release tag and passes the clone root as the build context, so the COPY
# apps/OpenSign/... lines below resolve). See opensign-client/README.md.
#
# The one deliberate deviation from upstream's Dockerfile is the
# `apt-get update && apt-get dist-upgrade` first layer. node:22.14.0 is a
# Debian (bookworm) image whose baked-in OS packages carry fixable
# CRITICAL/HIGH CVEs; this layer refreshes them so the workflow's Trivy scan
# (severity CRITICAL,HIGH, --exit-code 1) passes. The workflow also tracks the
# base image's registry digest (opensign-client/.opensign-client-base-digest)
# and rebuilds when Debian re-publishes node:22.14.0 with security fixes, so
# this dist-upgrade keeps the OS layer current on every rebuild.
FROM node:22.14.0

WORKDIR /usr/src/app

# Refresh the Debian OS packages (see the comment on the FROM above).
# --no-install-recommends keeps the image lean; the client needs no extra
# system libraries to run `npm install`, `npm run build` (Vite) or the plain
# Node static server (server.cjs).
RUN apt-get update \
  && apt-get -y --no-install-recommends dist-upgrade \
  && apt-get -y autoremove \
  && rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*

# Copy package.json + lockfile first to leverage the Docker build cache,
# then install (devDependencies included -- the upstream Dockerfile runs a
# plain `npm install`, not `npm ci --omit=dev`, so the Vite build tooling is
# present).
COPY apps/OpenSign/package*.json ./
RUN npm install

# The built SPA (Vite output lands in build/) plus the static file server and
# the entrypoint that injects REACT_APP_SERVERURL into a runtime env.js.
COPY apps/OpenSign/ .
COPY apps/OpenSign/.husky .
COPY apps/OpenSign/entrypoint.sh .
RUN chmod +x entrypoint.sh

ENV NODE_ENV=production
ENV GENERATE_SOURCEMAP=false

# Production build.
RUN npm run build

# Point the served index.html at the injected runtime env (the entrypoint
# creates env.js from REACT_APP_SERVERURL at container start).
RUN sed -i '/<head>/a\<script src="/env.js"></script>' build/index.html

# The static server (server.cjs) listens on 3000.
EXPOSE 3000

ENTRYPOINT ["./entrypoint.sh"]
CMD ["npm", "start"]
