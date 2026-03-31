# CoffeePotSocial — Technology Stack

## Frontend / Client
- **Ionic + Angular** — cross-platform app (Android, PWA/web from one codebase)
- **Capacitor** — native Android bridge
- **Angular** (standalone) — internal admin & moderation dashboard

## Backend
- **ASP.NET Core Web API** (.NET) — REST API, horizontally scaled
- **Docker + Docker Compose** — containerised, local dev parity

## Data
- **MongoDB** — primary database, DigitalOcean managed (3-node replica set); built-in text indexes for full-text post search
- **Redis** — feed cache, rate limit counters, feature flags

## Cloud & Infrastructure
- **DigitalOcean App Platform** — MVP hosting: free static sites (clientapp, admin), Docker container service for API (~$12/mo); DOKS (managed Kubernetes) post-MVP migration path
- **DigitalOcean** — managed MongoDB cluster
- **Cloudflare** — DNS, DDoS/WAF, CDN, edge caching
- **Cloudflare R2** — object/media storage (S3-compatible, zero egress)
- **Cloudflare Image Resizing** — on-demand image transforms (WebP/AVIF)
- **Terraform** — infrastructure as code for all manageable cloud resources (`infra/`); providers: `digitalocean`, `cloudflare`, `grafana`, `sentry`

## API & Auth
- **Zuplo** — API gateway (rate limiting, auth enforcement, security headers, versioning)
- **Firebase Auth** — identity provider (email/password, Google, Meta OAuth)
- **Firebase Cloud Messaging (FCM)** — push notifications (Android + PWA)
- **Google reCAPTCHA v3** — invisible bot protection

## CI/CD & DevOps
- **GitHub + GitHub Actions** — source control, full CI/CD pipeline
- **CodeQL · Semgrep · Gitleaks** — automated security scanning in CI
- **Firebase Emulator Suite** — local Auth and FCM emulation for development and testing
- **MinIO** — local S3-compatible storage emulating Cloudflare R2

## Observability
- **Sentry** — error tracking, performance monitoring, OTEL tracing
- **Grafana Cloud** (free tier) — unified observability: Loki (log aggregation), Prometheus metrics, Grafana dashboards
- **UptimeRobot** (free tier) — external uptime monitoring, public status page at status.coffeepotsocial.com

## External Services
- **Postmark** — transactional email
- **Beehiiv** — newsletter subscriber sync
- **Google Analytics** — client-side analytics (via Firebase)
- **Termly** — cookie consent, privacy policy, ToS compliance

## Media Processing
- **FFmpeg** — video transcoding (background worker)
- **HtmlSanitizer (.NET) + DOMPurify** — XSS prevention
