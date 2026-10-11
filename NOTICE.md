# NOTICE — Cap Screen Recording and Sharing

## Software deployed (not redistributed)

| Component | Source | License |
|---|---|---|
| Cap web app | `ghcr.io/capsoftware/cap-web@sha256:8ee4cbd3fd87f88f538831aed06c954c525db9c2426a62abeaf0ca307c5e1ce9`, from https://github.com/CapSoftware/Cap | AGPL-3.0 for the web app (repository `LICENSE`: AGPLv3 except the `cap-camera*`/`scap-*` crates, MIT, and third-party components under their own licenses) |
| Cap media server | `ghcr.io/capsoftware/cap-media-server@sha256:2dc90e055447026d7ff70656344cde74ec91b9d458a6050d9be6040a7074b62b` (same repository, `apps/media-server`) | AGPL-3.0 (same `LICENSE`); bundles ffmpeg |
| MySQL | `mysql:9.7.2` (Docker official image; Railway base definition) | GPL-2.0 (MySQL Community Server) |
| AWS CLI | `amazon/aws-cli:2.37.12` | Apache-2.0 |
| curl | `curlimages/curl:8.22.0` | curl license (MIT-style) |

The template references these images; it does not copy or modify their code.

## Upstream files checked (2026-10-11)

- `CapSoftware/Cap` (branch `main`): `docker-compose.yml`, `docker-compose.coolify.yml`, `docker-compose.template.yml` — services, variables, MySQL flags.
- `apps/web/Dockerfile` — `CMD HOSTNAME="0.0.0.0" node apps/web/server.js` (reason for the `::` start command).
- `apps/web/instrumentation.node.ts` — boot-time migrations (3 attempts, then exit) and bucket creation.
- `packages/env/server.ts` — variable names and defaults (`S3_PATH_STYLE`, `CAP_VIDEOS_DEFAULT_PUBLIC`, `CAP_ALLOWED_SIGNUP_DOMAINS`, AI keys).
- `packages/web-backend/src/S3Buckets/index.ts` — presigned URLs for media (private bucket compatible).
- `apps/media-server/src/index.ts`, `src/lib/auth.ts`, `src/lib/media-transfer.ts`, `railway.toml` — port, webhook secret, `MEDIA_SERVER_WEB_ORIGIN`.
- `apps/web/app/api/status/route.ts`, `apps/web/app/api/cron/*`, `apps/web/vercel.json` — health route, maintenance endpoints and Cap Cloud's schedules.
- `apps/web/content/docs/self-hosting.mdx`, `s3-config.mdx`, `s3-config/cloudflare-r2.mdx` — self-hosting steps, CORS with `ETag` exposed.
- `.github/workflows/docker-build-web.yml`, `docker-build-media-server.yml` — how the `latest` images are produced.
- GHCR tag lists for both images (only `latest`/`beta`/branch tags exist).

## Reused material

- **Railway base definition** `templates/_railway-base/mysql.json` (provided in this repository): variables and start command copied as instructed by STANDARD.md; image pinned to the patch release behind its `mysql:9` tag; upstream Cap flags appended; dedicated `cap` user/database added.
- **Round-1 template `04-plane`** (this project): the one-shot `put-bucket-cors` job pattern.
- **Market template `PwpGcf` "Cap"** (Railway marketplace, no license): used as the structural starting point (web + media server + MySQL). No README text or descriptions were copied.
- **Market template `cap-screen-recorder`** (no license): its technical findings were treated as hypotheses and re-verified against upstream source and a local run; no text or configuration copied.
- No `_upstream` repositories exist for this app. All prose and variable descriptions in this package are original.
