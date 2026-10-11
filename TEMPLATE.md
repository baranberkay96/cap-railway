# Cap Screen Recording and Sharing — Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Cap Screen Recording and Sharing** (no version label: Cap publishes the web server only as `latest`/`beta` builds, without version tags; see "Pinned versions") |
| Description | Record and share screen videos with links and comments. Loom alternative. (73 chars, unchanged from LISTINGS_ROUND2.md) |
| Category | Other |
| Tags | cap, screen-recording, video, loom-alternative, mysql, s3 |
| User input at deploy | **None.** All secrets are generated. Optional later: Resend (e-mail), Google sign-in, AI provider keys. |

## Base market template

- **`PwpGcf` "Cap"** (363 active / 91 recent projects, the template Cap's own docs link to) — the only widely used Cap web template. Kept: service split (web + media server + MySQL), `S3_PATH_STYLE=true`, private media-server URL and webhook URL.
  Fixed: `:latest` images, MinIO on a public domain with a **hardcoded root user** (`MINIO_ROOT_USER` is a fixed string in the template), a broken `${{secret(48}}` call, MySQL 8.0 (EOL April 2026) with a public TCP proxy and a `DATABASE_URL` pointing at that public proxy, no `MEDIA_SERVER_WEBHOOK_SECRET` (the media server rejects every callback without it), no DB wait (Cap gives up migrating after 3 tries), no healthchecks.
- **`cap-screen-recorder`** (2 projects, digest-pinned images, Railway Bucket) — technically the closest to this design. Its findings (path-style URLs for browser uploads, IPv4-only bind of the web image, shared webhook secret) were re-verified here against upstream source and in a local run; no text or config was copied.
- **`cap-1` (168 recent deploys) is not this product.** It deploys `tiago2/cap`, the image of **Cap, the proof-of-work CAPTCHA** (trycap.dev) with Valkey — a different open-source project that shares the name. It is not an alternative way to run CapSoftware's screen recorder, so nothing was taken from it. Its high deploy count reflects CAPTCHA demand, not difficulty with the official Cap images.
- No `_upstream` repos exist for this app. Market templates carry no license → reference only.

**Approach chosen:** official `ghcr.io/capsoftware/*` images, digest-pinned. They run correctly on Railway with two small adjustments (dual-stack bind, DB wait) done in the start command, so no custom build is needed.

## Pinned versions

| Service | Image | Tag / digest | Build date | Checked |
|---|---|---|---|---|
| Cap | `ghcr.io/capsoftware/cap-web` | `latest` = `sha256:8ee4cbd3fd87f88f538831aed06c954c525db9c2426a62abeaf0ca307c5e1ce9` (multi-arch index) | 2026-09-07 | 2026-10-11 |
| Cap Media Server | `ghcr.io/capsoftware/cap-media-server` | `latest` = `sha256:2dc90e055447026d7ff70656344cde74ec91b9d458a6050d9be6040a7074b62b` (multi-arch index) | 2026-10-10 | 2026-10-11 |
| MySQL | `mysql` | `9.7.2` (= Railway base `mysql:9` = MySQL 9.7 LTS) | 2026-09-29 | 2026-10-11 |
| Bucket CORS | `amazon/aws-cli` | `2.37.12` | 2026-10-09 | 2026-10-11 |
| Cap Maintenance | `curlimages/curl` | `8.22.0` | 2026-09-02 | 2026-10-11 |

Cap's GHCR packages only carry `latest`, `beta` (+ per-arch) for the web image and `latest` plus a few branch builds for the media server, so the digests are the pin. `cap-web` is built by a manually dispatched workflow (`docker-build-web.yml`); the media server is rebuilt on every push to `main` that touches `apps/media-server`. Both digests above are what `latest` resolved to on 2026-10-11, i.e. what Cap's own Coolify compose deploys today.

**Upgrade:** resolve the new `latest` digests (`docker buildx imagetools inspect ghcr.io/capsoftware/cap-web:latest`), put them on **Cap** and **Cap Media Server** together, back up MySQL, redeploy. Migrations run automatically when Cap boots.

## Architecture

```
Browser / Cap Desktop ──HTTPS──▶ Cap (Next.js, :3000, public, stateless)
        │                         │  boot: wait for MySQL → Drizzle migrations
        │                         ├─▶ MySQL 9.7 (accounts, videos, comments)        volume /var/lib/mysql
        │                         ├─▶ Cap Media Server (Bun + ffmpeg, :3456, private)
        │                         │      └── progress webhooks ──▶ Cap (private, shared secret)
        │                         └─▶ Bucket (presigned URLs)
        └── presigned PUT/GET ──────▶ Bucket (recordings, thumbnails; CORS rule set by Bucket CORS)
Cap Maintenance (cron */15) ──private──▶ Cap /api/cron/recover-failed-video-processing, /finalize-stale-desktop-segments
```

### Deviations from upstream compose (and why)

- **MinIO → Railway Bucket.** Upstream's compose runs MinIO (excluded by the standard). Cap talks plain S3 and serves media through presigned URLs (`packages/web-backend/src/S3Buckets`), which is exactly what private Railway Buckets support. `S3_PUBLIC_ENDPOINT` and `S3_INTERNAL_ENDPOINT` both point at the bucket endpoint (Buckets have no private endpoint). Cap's boot-time `CreateBucket`/public-read policy call fails harmlessly against the existing bucket (upstream catches it).
- **Bucket CORS job.** Browser uploads are multipart PUTs straight to the bucket and need `ETag` exposed (Cap's S3 docs). A one-shot `aws s3api put-bucket-cors` job sets that rule for Cap's domain (same pattern as the round-1 Plane template, which passed deploy tests).
- **Dual-stack bind.** The web image's `CMD` hard-codes `HOSTNAME="0.0.0.0"`. The start command sets `HOSTNAME=::`, so the media-server callbacks and the maintenance cron reach Cap over the private network in IPv6-only (legacy) environments as well. Verified locally: Next.js listens on `[::]:3000`. The media server (Bun) already listens on `[::]:3456`.
- **DB wait.** Cap migrates inside the web process 5 s after boot and calls `process.exit(1)` after three failed attempts. The start command waits up to 5 minutes for MySQL (BusyBox `nc`, present in the image), and the restart policy is `ALWAYS` as a second line of defence.
- **MySQL 9.7.2** (Railway base, LTS) instead of upstream's EOL `mysql:8.0`; Cap's migrations ran cleanly against 9.7.2 locally (42 tables created). Upstream's `--character-set-server`, `--collation-server`, `--max-connections` flags are kept; `mysql_native_password` is not needed (Cap's driver handles `caching_sha2_password`).
- **Maintenance cron.** Cap Cloud schedules `/api/cron/*` jobs on Vercel; a plain Docker install never calls them. Two of them are useful self-hosted: retrying failed/stalled video processing and finalizing stale desktop uploads. A Railway cron service calls them every 15 minutes with `CRON_SECRET` (Railway's minimum is 5 minutes; upstream runs them every 15). Marketing (Loops) and cloud-only jobs are not scheduled.

## Services

All services are created together from the template; references resolve at deploy time. Empty values are optional.

### Cap

- **Source:** Docker image `ghcr.io/capsoftware/cap-web@sha256:8ee4cbd3fd87f88f538831aed06c954c525db9c2426a62abeaf0ca307c5e1ce9`
- **Start command:** `/bin/sh -c "i=0; until nc -w 3 $MYSQL_WAIT_HOST 3306 </dev/null >/dev/null 2>&1; do i=$((i+1)); if [ $i -ge 150 ]; then echo 'MySQL not reachable after 5 minutes'; exit 1; fi; echo 'waiting for MySQL'; sleep 2; done; HOSTNAME=:: exec node apps/web/server.js"`
- **Healthcheck:** `/api/status` (timeout 600 s)
- **Public domain:** yes → target port `3000`
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ALWAYS
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3000` | Port the Next.js server listens on; Railway routes traffic and healthchecks here. |
| `WEB_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Public URL of this Cap instance (share links, desktop app sign-in, OAuth callbacks). Change it when you add a custom domain. |
| `NEXTAUTH_URL` | `${{WEB_URL}}` | Must equal WEB_URL; NextAuth builds its callback URLs from it. |
| `NEXTAUTH_SECRET` | `${{secret(64, "0123456789abcdef")}}` | Signs session tokens (32 random bytes, hex). Changing it signs everyone out. |
| `DATABASE_ENCRYPTION_KEY` | `${{secret(64, "0123456789abcdef")}}` | 32-byte hex key that encrypts stored credentials (e.g. a user's own S3 keys). Never change it after first boot. |
| `DATABASE_URL` | `mysql://${{MySQL.MYSQL_USER}}:${{MySQL.MYSQL_PASSWORD}}@${{MySQL.RAILWAY_PRIVATE_DOMAIN}}:3306/${{MySQL.MYSQL_DATABASE}}` | MySQL connection string over the private network (Cap requires MySQL). |
| `MYSQL_WAIT_HOST` | `${{MySQL.RAILWAY_PRIVATE_DOMAIN}}` | Host the start command waits for before booting; Cap migrates on boot and gives up after three quick attempts. |
| `CAP_AWS_BUCKET` | `${{Bucket.BUCKET}}` | Railway Bucket that stores recordings, thumbnails and exports. |
| `CAP_AWS_REGION` | `${{Bucket.REGION}}` | Bucket region. |
| `CAP_AWS_ACCESS_KEY` | `${{Bucket.ACCESS_KEY_ID}}` | Bucket access key. |
| `CAP_AWS_SECRET_KEY` | `${{Bucket.SECRET_ACCESS_KEY}}` | Bucket secret key. |
| `S3_PUBLIC_ENDPOINT` | `${{Bucket.ENDPOINT}}` | S3 endpoint used in the presigned URLs browsers and the desktop app upload to and play from. |
| `S3_INTERNAL_ENDPOINT` | `${{Bucket.ENDPOINT}}` | S3 endpoint for server-side calls (Railway Buckets have no private endpoint, so it equals the public one). |
| `S3_PATH_STYLE` | `true` | Use path-style bucket URLs (Cap's default for non-AWS storage). Keep true: browser uploads depend on the bucket CORS rule set by Bucket CORS. |
| `MEDIA_SERVER_URL` | `http://${{Cap Media Server.RAILWAY_PRIVATE_DOMAIN}}:3456` | Private address of the ffmpeg media server. |
| `MEDIA_SERVER_WEBHOOK_SECRET` | `${{secret(64, "0123456789abcdef")}}` | Shared secret for media-server progress callbacks; Cap Media Server references it. |
| `MEDIA_SERVER_WEBHOOK_URL` | `http://${{RAILWAY_PRIVATE_DOMAIN}}:3000` | Private address the media server calls back to (works because the start command binds to ::). |
| `CRON_SECRET` | `${{secret(64, "0123456789abcdef")}}` | Bearer token for Cap's maintenance endpoints; Cap Maintenance references it. |
| `CAP_VIDEOS_DEFAULT_PUBLIC` | `true` | New recordings are viewable by anyone with the link (upstream default). Set false to make them private until shared. |
| `CAP_ALLOWED_SIGNUP_DOMAINS` | *(empty)* | Comma-separated e-mail domains allowed to sign up, e.g. yourcompany.com. Empty = anyone who can receive the login code. |
| `RESEND_API_KEY` | *(empty)* | Resend API key for login-code and notification e-mails (HTTP API, works on every Railway plan). Without it, login codes are printed to this service's logs. |
| `RESEND_FROM_DOMAIN` | *(empty)* | Domain verified in Resend that e-mails are sent from, e.g. yourcompany.com. |
| `GOOGLE_CLIENT_ID` | *(empty)* | Optional Google OAuth client ID for 'Sign in with Google' (callback: WEB_URL/api/auth/callback/google). |
| `GOOGLE_CLIENT_SECRET` | *(empty)* | Optional Google OAuth client secret. |
| `ASSEMBLY_API_KEY` | *(empty)* | Optional AssemblyAI key: automatic transcription and captions. |
| `GROQ_API_KEY` | *(empty)* | Optional Groq key: AI titles, summaries and chapters (preferred generation provider). |
| `OPENAI_API_KEY` | *(empty)* | Optional OpenAI key: AI features fallback. |
| `ANTHROPIC_API_KEY` | *(empty)* | Optional Anthropic key: AI chat features. |

### Cap Media Server

- **Source:** Docker image `ghcr.io/capsoftware/cap-media-server@sha256:2dc90e055447026d7ff70656344cde74ec91b9d458a6050d9be6040a7074b62b`
- **Start command:** image default
- **Healthcheck:** `/health` (timeout 300 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ALWAYS
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3456` | Port the media server listens on (Bun binds [::], dual-stack); used for the healthcheck. |
| `MEDIA_SERVER_WEBHOOK_SECRET` | `${{Cap.MEDIA_SERVER_WEBHOOK_SECRET}}` | Shared with Cap: references Cap.MEDIA_SERVER_WEBHOOK_SECRET (edit it there). |
| `MEDIA_SERVER_WEB_ORIGIN` | `${{Cap.WEB_URL}}` | Cap's public origin; lets the media server fetch media through Cap's storage proxy with the shared secret. |

### MySQL

- **Source:** Docker image `mysql:9.7.2`
- **Start command:** `docker-entrypoint.sh mysqld --innodb-use-native-aio=0 --disable-log-bin --performance_schema=0 --innodb-buffer-pool-size=1G --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --max-connections=1000`
- **Healthcheck:** none
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/var/lib/mysql`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `MYSQLHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Railway Private Domain Name. |
| `MYSQLPORT` | `3306` | MySQL port. |
| `MYSQLUSER` | `root` | MySQL user, used for the Data panel. |
| `MYSQL_URL` | `mysql://${{ MYSQLUSER }}:${{ MYSQL_ROOT_PASSWORD }}@${{ RAILWAY_PRIVATE_DOMAIN }}:3306/${{ MYSQL_DATABASE }}` | URL to connect to MySQL. |
| `MYSQLDATABASE` | `${{ MYSQL_DATABASE }}` | Default database, used for Data panel. |
| `MYSQLPASSWORD` | `${{ MYSQL_ROOT_PASSWORD }}` | Root password, used for Data panel. |
| `MYSQL_DATABASE` | `cap` | Database created on first boot; Cap's DATABASE_URL points at it. |
| `MYSQL_ROOT_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Root password for MySQL DB. |
| `MYSQL_USER` | `cap` | Application user created on first boot with full rights on MYSQL_DATABASE. |
| `MYSQL_PASSWORD` | `${{secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Password of the application user (letters and digits only, it is embedded in a URL). |

### Bucket CORS

- **Source:** Docker image `amazon/aws-cli:2.37.12`
- **Start command:** `/bin/sh -c "aws s3api put-bucket-cors --bucket $BUCKET_NAME --endpoint-url $S3_ENDPOINT --region $S3_REGION --cors-configuration \"$CORS_CONFIGURATION\" && echo 'Bucket CORS configured for' $PUBLIC_ORIGIN"`
- **Healthcheck:** none
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** NEVER
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | `${{Bucket.ACCESS_KEY_ID}}` | Bucket access key. |
| `AWS_SECRET_ACCESS_KEY` | `${{Bucket.SECRET_ACCESS_KEY}}` | Bucket secret key. |
| `BUCKET_NAME` | `${{Bucket.BUCKET}}` | Bucket to configure. |
| `S3_ENDPOINT` | `${{Bucket.ENDPOINT}}` | Bucket S3 endpoint. |
| `S3_REGION` | `${{Bucket.REGION}}` | Bucket region. |
| `PUBLIC_ORIGIN` | `https://${{Cap.RAILWAY_PUBLIC_DOMAIN}}` | Cap's public origin (used in the log line). |
| `CORS_CONFIGURATION` | `{"CORSRules":[{"AllowedOrigins":["https://${{Cap.RAILWAY_PUBLIC_DOMAIN}}"],"AllowedMethods":["GET","HEAD","PUT","POST"],"AllowedHeaders":["*"],"ExposeHeaders":["ETag"],"MaxAgeSeconds":3600}]}` | CORS rule letting browsers on Cap's domain upload (multipart, needs ETag) and play recordings via presigned URLs. Add custom domains to AllowedOrigins and redeploy this service. |

### Cap Maintenance

- **Source:** Docker image `curlimages/curl:8.22.0`
- **Start command:** `/bin/sh -c "for job in recover-failed-video-processing finalize-stale-desktop-segments; do curl -sS --max-time 240 --retry 2 --retry-delay 10 --retry-connrefused -o /dev/null -w \"$job: HTTP %{http_code} in %{time_total}s\n\" -H \"Authorization: Bearer $CRON_SECRET\" \"$CAP_INTERNAL_URL/api/cron/$job\"; done"`
- **Cron schedule:** `*/15 * * * *` (runs to completion)
- **Healthcheck:** none
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** NEVER
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `CAP_INTERNAL_URL` | `http://${{Cap.RAILWAY_PRIVATE_DOMAIN}}:3000` | Cap's private address. |
| `CRON_SECRET` | `${{Cap.CRON_SECRET}}` | Shared with Cap: references Cap.CRON_SECRET (edit it there). |

### Bucket (Railway Bucket)

Create a Bucket named `Bucket` (any region). Cap, Bucket CORS reference `BUCKET`, `ENDPOINT`, `REGION`, `ACCESS_KEY_ID` and `SECRET_ACCESS_KEY` from it.


## Improvements over market templates

| # | Change | Why |
|---|---|---|
| 1 | Official images pinned by digest (leader: `:latest`) | Reproducible deploys; Cap has no version tags. |
| 2 | Railway Bucket + one-shot CORS job instead of a public MinIO with a hardcoded root user | No credentials in the template, managed storage, free bucket egress for playback. |
| 3 | `MEDIA_SERVER_WEBHOOK_SECRET` generated and shared by reference | Without it the media server rejects every call (`validateMediaServerSecret`), so processing never finishes. |
| 4 | Web server bound to `::` | Media callbacks and cron reach Cap in IPv6-only environments too. |
| 5 | MySQL wait in the start command + `ALWAYS` restart | Cold deploys no longer race the database (Cap exits after 3 migration attempts). |
| 6 | MySQL 9.7.2 LTS, private only, dedicated `cap` user | Supported version; no public TCP proxy; app does not use root. |
| 7 | Healthchecks: `/api/status` (web), `/health` (media) | Railway only routes to a healthy deployment. |
| 8 | Cap Maintenance cron | Failed or stalled processing is retried automatically, as on Cap Cloud. |
| 9 | Optional Resend, Google sign-in, AI and signup-domain variables pre-declared with descriptions | Mail, SSO and AI are a form away; nothing is required to boot. |

## First login

1. Open the public URL, enter your e-mail and request a login code.
2. Without `RESEND_API_KEY`, the 6-digit code is printed in the **Cap** service logs (Railway → Cap → Logs). Enter it to sign in. Anyone who can read your Railway logs can sign in this way, so set up Resend before inviting people.
3. Optional: set `RESEND_API_KEY` + `RESEND_FROM_DOMAIN` (a domain verified in Resend), and `CAP_ALLOWED_SIGNUP_DOMAINS` to your company domain; redeploy Cap.
4. Cap Desktop / mobile: in settings, set the server URL to the public URL.
5. Custom domain: add it to **Cap**, update `WEB_URL`, add the origin to `CORS_CONFIGURATION` on **Bucket CORS**, redeploy Cap and Bucket CORS.

## Scaling

- **Cap (web):** stateless, but migrations run inside the web process at boot. Keep 1 replica unless you accept concurrent migration attempts on upgrades; scale vertically first.
- **Cap Media Server:** transcoding is CPU-bound and bursty. Give it more vCPU/RAM; it reads its cgroup limits and caps concurrent jobs itself (`MEDIA_SERVER_MAX_CONCURRENT_VIDEO_PROCESSES` to override). Several replicas behind the one private hostname are possible but untested.
- **MySQL:** vertical; Railway can convert it to an HA cluster (MySQL 8/9 supported).

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month, Bucket $0.015/GB-month. Idle figures measured locally (2026-10-11) on the pinned images.

| Service | Idle RAM | Idle CPU | Typical RAM (10-person team) | Typical CPU |
|---|---|---|---|---|
| Cap | 300 MB | 0.01 vCPU | 450 MB | 0.05 vCPU |
| Cap Media Server | 100 MB | <0.01 vCPU | 500 MB avg (1–2 GB peaks while transcoding) | 0.2 vCPU avg (1–2 vCPU bursts) |
| MySQL | 270 MB | 0.01 vCPU | 350 MB | 0.02 vCPU |
| Bucket CORS, Cap Maintenance | 0 (exit) | 0 | 0 | 0 |
| **Total** | **≈0.67 GB** | **≈0.03 vCPU** | **≈1.3 GB** | **≈0.27 vCPU** |

- **Idle:** ≈ $6.7 RAM + $0.6 CPU + $0.15 volume ≈ **$7–8/month**.
- **Typical:** ≈ $13 RAM + $5.4 CPU + $0.3 volume + Bucket 100 GB ($1.5) ≈ **$20/month**, plus service egress for transcoded files the media server uploads back to the bucket. Cost tracks recording volume, not uptime.
- **Minimum plan:** Hobby. Idle fits Trial's 1 GB/service, but transcoding peaks exceed it.

## Verification checklist

- [ ] **Bucket CORS** logs `Bucket CORS configured for https://…` and exits 0.
- [ ] **Cap** logs `waiting for MySQL` (cold deploy only), then `💿 Migrations run successfully!`; deployment healthy on `/api/status`.
- [ ] **Cap Media Server** healthy on `/health` (`"status":"ok"`, ffmpeg available).
- [ ] Sign in with the code from the Cap logs.
- [ ] Record or upload a short video in the browser: no CORS error in the browser console, the object appears in the Bucket, the share page plays it, the thumbnail appears (proves media-server processing and the private webhook).
- [ ] Connect Cap Desktop to the URL and upload a recording.
- [ ] **Cap Maintenance** (trigger once manually) logs `recover-failed-video-processing: HTTP 200` and `finalize-stale-desktop-segments: HTTP 200`.
- [ ] Redeploy Cap: videos and accounts still there.

## Open questions

- **Bucket CORS / URL style:** Railway documents virtual-hosted URLs for new buckets and does not document CORS. Cap defaults to path-style; the CORS job sets the rule explicitly. Verified on Railway by the round-1 Plane template (put-bucket-cors succeeded) but not yet with Cap's multipart upload; checklist item above.
- **Workflow state:** Cap's video pipeline uses the Workflow DevKit with its default local world. Its state does not survive a Cap restart; the maintenance cron's recovery job is upstream's mechanism for restarting such runs. Not exercised end-to-end here.
- **Image skew:** the web digest (2026-09-07) is older than the media-server digest (2026-10-10) because upstream rebuilds them on different triggers; this matches what `latest` gives every Cap self-hoster today. The local smoke test (health, webhook reachability, cron endpoints, migrations) passed with this pair.
- **Railway start-command parsing** of the `/bin/sh -c "…"` strings was simulated with shell-style splitting and executed against the real images; same pattern as round-1 templates that passed deploy tests.
