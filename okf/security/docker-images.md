---
type: Security Reference
title: "Docker image supply chain — archive_letters stack"
description: Provenance, tagging, and exposure assessment of the images the archive_letters paperless-ngx stack runs, with hardening notes.
tags: [security, supply-chain, docker, archive-letters]
status: stable
stale_after: 2026-12-19
generated: { by: process:archive_assistant, at: 2026-09-19T15:45:00Z }
sources:
  - id: compose
    resource: ../../docker-compose.yml  # archive_letters deployment dir (outside this repo clone)
    title: archive_letters deployment compose
    author: human:tronco
  - id: ghcr-pkg
    resource: https://github.com/orgs/paperless-ngx/packages/container/package/paperless-ngx
    title: Official paperless-ngx image package (ghcr.io)
    author: paperless-ngx/maintainers
  - id: docker-library
    resource: https://github.com/docker-library/official-images
    title: Docker official-images (redis, postgres)
    author: docker-library/official-images
---

# Images in use

| Service | Image | Publisher | Role |
|---|---|---|---|
| `webserver` | `ghcr.io/paperless-ngx/paperless-ngx:latest` | paperless-ngx project org (GitHub official package) | app + OCR (Tesseract) + Tika extractor, s6-supervised |
| `db` | `docker.io/library/postgres:16-alpine` | Docker Official Images (docker-library) | PostgreSQL 16 |
| `broker` | `docker.io/library/redis:7-alpine` | Docker Official Images (docker-library) | Redis 7 — task/Cel broker |

# Assessment

**Provenance — good.** All three come from well-known, curated publishers:
the paperless-ngx *project's own* package on GitHub Container Registry (where
the project's CI builds the image), and the Docker Official Images for
Redis/Postgres (multi-arch builds maintained by the docker-library
process).[^ghcr-pkg][^docker-library] This stack does **not** pull images
from third-party reskin registries — that is the main supply-chain trap with
app stacks like this and it is avoided here.

**Tagging — the one real weakness.** The app image is pinned to `latest`,
a floating tag: a `docker compose pull && up` at any date can change the
running bits (features, CVE patches, *and* regressions). That is acceptable
for a dev/archive box, but it breaks reproducibility and turns every
re-pull into an unreviewed change.[^compose] Recommended hardening, in
order of value:

1. **Pin the app image to a versioned tag** (e.g. a current `2026.09.x`-style
   release tag) or, for full reproducibility, an image digest
   (`@sha256:...`) — then changes are deliberate and diffable in git.
2. Keep `postgres:16-alpine` / `redis:7-alpine` major-pinned (already are —
   minor upgrades flow in; that is normal and fine).
3. Track the project's release notes/security advisories before bumping the
   pinned tag.

**Exposure — minimal.** Only the app publishes a port: `8000` (bound to all
interfaces of this LAN box; the DB and Redis expose **no** host ports — they
are reachable only on the compose-internal network). Data/consume/media
volumes are host directories owned by UID/GID 1000 (`USERMAP_UID/GID`),
which the app container maps to its internal user — a sensible non-root
posture for a single-user LAN deployment.

**Residual notes.**
- The DB password lives in the compose file — acceptable for a local archive
  stack on a trusted host; do not copy this file into any public repository.
- `alpine` bases are smaller (less surface) but use musl; for Redis/Postgres
  this is a non-issue in practice.
- The compose file declares the obsolete `version:` key (harmless; compose
  warns and ignores it).

# Bottom line

For a single-user LAN archive: **the image set is from trustworthy sources
with minimal network exposure.** The only meaningful hardening step is
pinning the paperless image off `latest` (tag or digest) so every change is
reviewable in version control.

[^compose]: archive_letters deployment compose
[^ghcr-pkg]: Official paperless-ngx image package (ghcr.io)
[^docker-library]: Docker official-images (redis, postgres)
