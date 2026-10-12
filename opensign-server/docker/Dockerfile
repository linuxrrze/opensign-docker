# OpenSign backend API (Node/Express + the Parse.js layer + Mailgun/SMTP/
# DoSpace integrations + LibreOffice DOCX -> PDF conversion).
#
# Mirrors upstream's apps/OpenSignServer/Dockerhubfile -- same node:22.14.0
# base, same `apt-get install -y libreoffice`, same npm install + node
# index.js -- but the build runs from a fresh clone of the monorepo (the
# workflow checks out the tracked release tag and passes the clone root as the
# build context, so the COPY apps/OpenSignServer/... lines below resolve). See
# opensign-server/README.md.
#
# The one deliberate deviation from upstream's Dockerfile is the
# `apt-get update && apt-get -y dist-upgrade` first. node:22.14.0 is a Debian
# (bookworm) image whose baked-in OS packages carry fixable CRITICAL/HIGH
# CVEs; this layer refreshes them so the workflow's Trivy scan (severity
# CRITICAL,HIGH, --exit-code 1) passes. The workflow also tracks the base
# image's registry digest (opensign-server/.opensign-server-base-digest) and
# rebuilds when Debian re-publishes node:22.14.0 with security fixes, so this
# dist-upgrade keeps the OS layer current on every rebuild.
#
# libreoffice is installed with its recommended packages (no
# --no-install-recommends, exactly like upstream) so the DOCX -> PDF filters
# are complete.
FROM node:22.14.0

# Refresh the Debian OS packages, then install LibreOffice for DOCX -> PDF
# conversion (the heaviest step). All in one layer so the apt metadata is
# written and wiped in the same RUN (no stale layer).
RUN apt-get update \
  && apt-get -y dist-upgrade \
  && apt-get install -y libreoffice \
  && apt-get -y autoremove \
  && rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*

WORKDIR /usr/src/app

# Copy package.json + lockfile first to leverage the Docker build cache, then
# install (devDependencies included -- the upstream Dockerfile runs a plain
# `npm install`, not `npm ci --omit=dev`).
COPY apps/OpenSignServer/package*.json ./
RUN npm install

# The API (index.js + the Parse.js layer, cloud code, etc.).
COPY apps/OpenSignServer/ .

# The Parse.js API listens on 8080 and mounts its endpoints at $PARSE_MOUNT
# (/app by default).
EXPOSE 8080

CMD ["npm", "start"]
