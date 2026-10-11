# Deploy and Host Cap with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/cap-screen-recording-and-sharing?referralCode=20H68q)

Cap is an open-source screen recorder: record your screen and camera, get a shareable link right away, and collect comments on the video. This community template runs Cap's web app and ffmpeg media server on Railway with MySQL and a Railway Bucket for the recordings, so your videos stay in your own project.

## About Hosting Cap

Cap is a Next.js web app that hosts the dashboard, share pages and the API used by Cap Desktop, plus a media server that runs ffmpeg for transcoding, thumbnails and previews. It needs MySQL for accounts and video metadata and S3-compatible storage for the files, which browsers and the desktop app upload to directly. This template uses Cap's official images pinned by digest, a Railway Bucket with the CORS rule browser uploads need, a private media server with a generated callback secret, a database wait so first boot does not race MySQL, and a small cron job that retries failed video processing. All secrets are generated at deploy time.

## Common Use Cases

- Async updates and stand-ups recorded in a few minutes instead of long written reports
- Bug reports that show the problem on screen, linked from your issue tracker
- Onboarding, support and how-to walkthroughs on a URL you control
- Recordings of customer or internal systems that must not leave your own infrastructure

## Dependencies for Cap Hosting

- Cap web app (`ghcr.io/capsoftware/cap-web`, digest-pinned)
- Cap media server with ffmpeg (`ghcr.io/capsoftware/cap-media-server`, digest-pinned)
- MySQL (Railway MySQL 9.7)
- S3-compatible object storage (Railway Bucket)
- Optional: a Resend account for login and notification e-mails

### Deployment Dependencies

- Cap self-hosting guide: https://cap.so/docs/self-hosting
- Cap source and Docker Compose files: https://github.com/CapSoftware/Cap
- Environment variable reference: https://github.com/CapSoftware/Cap/blob/main/packages/env/server.ts
- Resend (e-mail API): https://resend.com
- Railway Buckets: https://docs.railway.com/storage-buckets

### Implementation Details

| Service | Image | Role |
|---|---|---|
| Cap | `ghcr.io/capsoftware/cap-web` (digest) | Web app and API on the public domain; runs database migrations at boot |
| Cap Media Server | `ghcr.io/capsoftware/cap-media-server` (digest) | ffmpeg processing, private network only |
| MySQL | `mysql:9.7.2` | Accounts, videos, comments (volume, no public proxy) |
| Bucket | Railway Bucket | Recordings, thumbnails, exports |
| Bucket CORS | `amazon/aws-cli:2.37.12` | One-shot job that sets the bucket CORS rule for Cap's domain |
| Cap Maintenance | `curlimages/curl:8.22.0` | Cron every 15 minutes: retries failed or stalled video processing |

**First login:** open the public URL and enter your e-mail. Until you add a Resend API key, the 6-digit login code is written to the Cap service logs; copy it from there. Before inviting others, set `RESEND_API_KEY` and `RESEND_FROM_DOMAIN` (Resend is an HTTP API, so it works on every Railway plan) and optionally restrict sign-ups with `CAP_ALLOWED_SIGNUP_DOMAINS`. In Cap Desktop, set the server URL to `https://{your-domain}`.

**Custom domain:** add it to the Cap service, update `WEB_URL`, add `https://{your-domain}` to `CORS_CONFIGURATION` on Bucket CORS, and redeploy both.

**Scaling:** give Cap Media Server more CPU and memory if many recordings are processed at once; it adapts its concurrency to the resources it sees. Keep Cap at one replica (it migrates at boot) and scale it vertically.

**Versions:** Cap publishes rolling `latest` images, so this template pins image digests. To upgrade, set new digests on Cap and Cap Media Server together, back up MySQL, and redeploy; migrations run on boot.

### Why Deploy Cap on Railway?

Railway runs the web app, media server, database and bucket as one project with private networking, managed volumes and usage-based billing. Video playback and uploads go straight to the bucket, so you get a self-hosted Loom alternative without managing servers, certificates or storage yourself.
