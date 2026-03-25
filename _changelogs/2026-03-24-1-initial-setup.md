# 2026-03-24 — Initial Project Setup & Specification

## Summary
First session. No code written yet — this entry covers all planning, architecture, and spec decisions made before development began.

## Changes

### SPEC.md — Created
- Authored full project specification covering 15 sections: overview, goals, features, architecture, tech stack, data models, API design, auth, media pipeline, push notifications, analytics, newsletter sync, monitoring, admin tooling, scalability, security, NFRs, and open questions

### Architecture & Tech Stack — Locked
- **Frontend:** Ionic + Angular (`clientapp/`)
- **Admin:** Angular standalone (`admin/`)
- **Backend:** ASP.NET Core Web API (`api/`)
- **Database:** MongoDB on DigitalOcean managed cluster
- **Cache:** Redis (feed cache, rate limit counters)
- **Auth:** Firebase Auth (email/password, Google, Meta OAuth)
- **Push:** Firebase Cloud Messaging (FCM)
- **Analytics:** Google Analytics via Firebase SDK
- **Storage:** Cloudflare R2 (S3-compatible, zero egress, presigned PUT URLs)
- **Image processing:** Cloudflare Image Resizing (on-demand URL transforms at CDN layer — no in-process library)
- **DNS / CDN / Edge:** Cloudflare
- **API Gateway:** Zuplo
- **IaC:** Terraform
- **Monitoring:** Sentry
- **CI/CD:** GitHub Actions
- **Containers:** Docker + Docker Compose
- **Compliance / consent:** Termly
- **Transactional email:** Postmark
- **Newsletter:** Beehiiv (opt-in subscriber sync via `NewsletterSyncWorker`)

### Data Model — Defined
- MongoDB document schemas for: `users`, `posts`, `media`, `follows`, `likes`, `notifications`, `reports`, `auditLog`, `featureFlags`
- `users` document includes `newsletterOptIn` and `newsletterSynced` flags for Beehiiv export tracking

### Newsletter Opt-In — Designed
- Pre-checked signup checkbox: "Also add me to the Coffee Pot Newsletter"
- `NewsletterSyncWorker` (IHostedService) polls for unsynced opt-ins and pushes to Beehiiv API

### ARCHITECTURE.mmd — Created
- Full Mermaid flowchart reflecting the locked architecture: clients → Cloudflare edge → Zuplo → ASP.NET Core → data/storage/Firebase/external services
