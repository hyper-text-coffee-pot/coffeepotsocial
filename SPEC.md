# CoffeePotSocial — Project Specification

> Living document. Updated as decisions are made.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [Core Features (MVP)](#3-core-features-mvp)
4. [Architecture Overview](#4-architecture-overview)
5. [Tech Stack](#5-tech-stack)
6. [Data Models](#6-data-models)
7. [API Design](#7-api-design)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [Media Processing Pipeline](#9-media-processing-pipeline)
10. [Monitoring & Observability](#10-monitoring--observability)
11. [Admin Tooling](#11-admin-tooling)
12. [Scalability & Infrastructure](#12-scalability--infrastructure)
13. [Security](#13-security)
14. [Non-Functional Requirements](#14-non-functional-requirements)
15. [Open Questions](#15-open-questions)

---

## 1. Project Overview

CoffeePotSocial is an open-source, Twitter-like social network built live and in public with AI assistance. The project is as much about the *process* of building as it is the end product — demonstrating that a real, production-grade social platform can be built with AI as a development partner.

**Core experience:** Users can post short messages, follow other users, and see a feed of content from the people they follow.

### Repository Structure

```
coffeepotsocial/
├── clientapp/      — Ionic + Angular: mobile & web client (Android, PWA)
├── admin/          — Angular standalone: internal admin & moderation dashboard
├── api/            — ASP.NET Core Web API: backend REST API
├── actions/        — GitHub Actions workflow definitions
├── _changelogs/    — Release changelogs
├── SPEC.md         — This document
├── ARCHITECTURE.mmd— Mermaid architecture diagram
├── README.md
└── LICENSE
```

---

## 2. Goals & Non-Goals

### Goals

- Build a functional, self-hostable social network
- Keep the codebase clean, well-tested, and open to contribution
- Make architectural decisions transparently and document them here
- Ship iteratively — working software over comprehensive planning

### Non-Goals (for MVP)

- Replacing Twitter/X at scale
- Monetisation features
- Advanced moderation tooling (basic moderation only at first)
- ActivityPub / Fediverse federation (long-term consideration, not MVP)

---

## 3. Core Features (MVP)

### Users
- [ ] Register an account (email + password)
- [ ] Log in / log out
- [ ] Public profile page (username, display name, bio, avatar)
- [ ] Follow / unfollow other users
- [ ] Follower / following counts

### Posts ("Brews")
- [ ] Create a post (text, max 280 characters — subject to change)
- [ ] Delete own posts
- [ ] View a single post's permalink page
- [ ] Feed of posts from followed users (chronological, to start)

### Interactions
- [ ] Like a post
- [ ] Reply to a post (threaded)
- [ ] Repost (simple repost, no quote-post for MVP)

### Discovery
- [ ] Global/public timeline
- [ ] Basic user search (by username)

### Notifications
- [ ] In-app notifications for: likes, replies, follows, reposts

---

## 4. Architecture Overview

> High-level. Will be refined as infrastructure decisions are finalised.

```
                   ┌────────────────────────────────────────┐
                   │             Cloudflare Edge             │
                   │  DNS · DDoS protection · CDN · WAF     │
                   │  Edge caching for static assets/media  │
                   └──────────────────┬─────────────────────┘
                                      │ HTTPS
                   ┌──────────────────▼─────────────────────┐
                   │               Zuplo                    │
                   │          API Gateway Layer             │
                   │  Rate limiting · Auth enforcement      │
                   │  Security headers · API key mgmt       │
                   │  Request routing · API versioning      │
                   └──────────────────┬─────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
   ┌──────────▼──────┐    ┌───────────▼─────┐    ┌──────────▼──────┐
   │  ASP.NET Core   │    │  ASP.NET Core   │    │  ASP.NET Core   │
   │  Web API (1)    │    │  Web API (2)    │    │  Web API (n)    │
   │  [Docker]       │    │  [Docker]       │    │  [Docker]       │
   └──────┬──────────┘    └──────┬──────────┘    └──────┬──────────┘
          └──────────────────────┼──────────────────────┘
                                 │
          ┌──────────────────────┼─────────────────────┐
          │                      │                     │
┌─────────▼──────┐    ┌──────────▼─────┐
│   MongoDB      │    │     Redis      │
│  (DigitalOcean │    │  Cache/Sessions│
│   Managed      │    │  Rate counters │
│   Cluster)     │    │  Feed cache    │
└────────────────┘    └────────────────┘

        ┌──────────────────────────┐     ┌───────────────────────┐
        ┌────────────────────────────────────────────────────┐
        │              Cloudflare R2                         │
        │  S3-compatible · Zero egress · Presigned PUT URLs  │
        │  + Cloudflare Image Resizing (on-demand transforms)│
        └────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                     Client Layer                             │
│  ┌─────────────────────┐     ┌──────────────────────────┐   │
│  │   Ionic Web / PWA   │     │  Ionic Android App       │   │
│  │   (browser)         │     │  (Capacitor compiled)    │   │
│  └─────────────────────┘     └──────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                   Observability Stack                        │
│  Errors + Perf → Sentry  │  Metrics → DO monitoring + Grafana  │
│  Traces → Sentry OTEL    │  Alerts → Sentry alerting rules     │
│  Logs → aggregator TBD   │  Uptime → external monitor          │
└──────────────────────────────────────────────────────────────┘
```

### Key Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Edge / DNS** | Cloudflare | DDoS protection, global CDN, WAF, zero-config HTTPS, free tier covers MVP |
| **API Gateway** | Zuplo | Rate limiting, auth enforcement, and API versioning without reinventing the wheel in application code |
| **Backend** | ASP.NET Core Web API (.NET) | Mature, high-performance, strong typing, excellent MongoDB driver and ecosystem |
| **Database** | MongoDB on DigitalOcean | Flexible document model suits social data well; DO managed cluster handles replication and backups |
| **Frontend / Mobile** | Ionic | Single codebase targets web (PWA), Android, and iOS — hybrid native via Capacitor |
| **Monolith vs. services** | Monolith to start | Avoid distributed systems complexity pre-scale; extract if/when warranted |
| **Feed strategy** | Fan-out on read (pull) for MVP | Simple to implement; fan-out on write if follower counts demand it |
| **Async work** | In-process background tasks (`IHostedService`) for MVP; DigitalOcean Managed Kafka if workload demands a queue post-MVP | Simple to start; Kafka available on DO if needed |
| **Object storage** | Cloudflare R2 (S3-compatible, zero egress) | All media in one store; S3 presigned PUT URLs; zero egress fees; Cloudflare CDN built-in; Cloudflare Stream available post-MVP for video |
| **Real-time** | Polling for MVP → SSE/WebSockets post-MVP | Capacitor supports native WebSocket; SSE fine for web |
| **Orchestration** | DigitalOcean App Platform or Kubernetes \u2014 TBD | App Platform is simpler; Kubernetes gives more control; decide after MVP |

---

## 5. Tech Stack

> Locked decisions are reflected throughout this spec. Open items are tracked in §15.

| Layer | Decision | Status |
|---|---|---|
| **DNS & Edge** | Cloudflare (DNS, DDoS protection, CDN, edge caching) | Locked |
| **API Gateway** | Zuplo (rate limiting, auth enforcement, security headers, API key mgmt) | Locked |
| **Client app** (`clientapp/`) | Ionic + Angular — mobile & web client | Locked |
| **Admin app** (`admin/`) | Angular standalone — internal admin & moderation dashboard | Locked |
| **Backend** (`api/`) | ASP.NET Core Web API (.NET) | Locked |
| **Mobile targets** | Android (Capacitor); iOS post-MVP; PWA | Locked |
| **Database** | MongoDB on DigitalOcean (managed cluster) | Locked |
| **Cache** | Redis (feed cache, rate limit counters) | Locked |
| **Object / blob storage** | Cloudflare R2 (S3-compatible API, zero egress) | Locked |
| **CDN for media** | Cloudflare CDN (native to R2, no separate config needed) | Locked |
| **Job queue** | None for MVP — DigitalOcean Managed Kafka if async processing is needed post-MVP | Deferred |
| **Auth — identity provider** | Firebase Auth (email/password, Google, Meta) | Locked |
| **Push notifications** | Firebase Cloud Messaging (FCM) | Locked |
| **Analytics** | Google Analytics (via Firebase) | Locked |
| **Image processing** | Cloudflare Image Resizing (on-demand URL-based transforms served via CF CDN from R2) | Locked |
| **Video transcoding** | FFmpeg in worker containers | Locked pattern |
| **Source control** | GitHub | Locked |
| **CI/CD** | GitHub Actions | Locked |
| **Containerisation** | Docker + Docker Compose | Locked |
| **Orchestration (production)** | DigitalOcean App Platform vs. Kubernetes — TBD | Open |
| **IaC** | Terraform | Locked |
| **Monitoring** | Sentry (errors, performance, alerting) | Locked |
| **Compliance & consent** | Termly (privacy policy, cookie consent, ToS) | Locked |
| **Metrics / dashboards** | DigitalOcean native monitoring + Grafana (TBD) | Open |
| **Email (transactional)** | Postmark | Locked |
| **Newsletter** | Beehiiv (subscriber sync via API) | Locked |

---

## 6. Data Models

> MongoDB document schemas. `_id` is `ObjectId` unless noted. Fields marked `// embedded` are subdocuments; fields marked `// ref` are ObjectId references to another collection. Exact field names and indexes will be defined in code — this is a design-level reference.

### Collection: `users`
```js
{
  _id: ObjectId,
  firebaseUid: String,        // unique index — stable Firebase Auth UID
  username: String,           // unique index
  displayName: String,
  email: String,              // unique index (sourced from Firebase)
  bio: String,
  avatarUrl: String,          // CDN URL of processed avatar
  role: String,               // enum: "user" | "moderator" | "admin"
  suspended: Boolean,         // default: false
  suspendedAt: Date,
  followerCount: Number,      // denormalised counter
  followingCount: Number,     // denormalised counter
  fcmTokens: [String],        // FCM device tokens for push notifications (array for multi-device)
  newsletterOptIn: Boolean,   // true if user opted in on signup (checkbox pre-checked); default: true
  newsletterSynced: Boolean,  // false until exported to newsletter API by sync worker; default: false
  createdAt: Date,
  updatedAt: Date
}

// Indexes:
//   { firebaseUid: 1 }  unique  — primary lookup on every auth'd request
//   { username: 1 }     unique
//   { email: 1 }        unique
//   { createdAt: -1 }
//   text index on { username, displayName }  — for search
```

> The `oauthAccounts` collection from the earlier schema is **removed** — provider linking is managed by Firebase Auth, not our database.

### Collection: `posts`
```js
{
  _id: ObjectId,
  authorId: ObjectId,         // ref: users
  body: String,               // max 280 chars; nullable for media-only posts
  replyToId: ObjectId,        // ref: posts — nullable
  repostOfId: ObjectId,       // ref: posts — nullable
  media: [                    // embedded array (up to 4 items)
    {
      mediaId: ObjectId,      // ref: media
      position: Number,       // 0-indexed
      cdnUrl: String,         // denormalised for fast reads
      type: String            // "image" | "video"
    }
  ],
  likeCount: Number,          // denormalised counter
  replyCount: Number,         // denormalised counter
  repostCount: Number,        // denormalised counter
  hidden: Boolean,            // default: false — moderator action
  createdAt: Date
}

// Indexes:
//   { authorId: 1, createdAt: -1 }     — user timeline
//   { createdAt: -1 }                   — public timeline
//   { replyToId: 1, createdAt: 1 }     — reply threads
//   text index on { body }             — full-text post search
```

### Collection: `media`
```js
{
  _id: ObjectId,
  uploaderId: ObjectId,       // ref: users
  type: String,               // "image" | "video"
  status: String,             // "pending" | "processing" | "ready" | "failed"
  storageKey: String,         // path in object storage
  cdnUrl: String,             // available once status = "ready"
  contentType: String,        // e.g. "image/webp", "video/mp4"
  byteSize: Number,
  durationSecs: Number,       // nullable — videos only
  width: Number,
  height: Number,
  createdAt: Date
}

// Indexes:
//   { uploaderId: 1, createdAt: -1 }
//   { status: 1 }  — for worker polling / cleanup jobs
```

### Collection: `follows`
```js
{
  _id: ObjectId,
  followerId: ObjectId,       // ref: users
  followingId: ObjectId,      // ref: users
  createdAt: Date
}

// Indexes:
//   { followerId: 1, followingId: 1 }  unique
//   { followerId: 1, createdAt: -1 }   — who does this user follow?
//   { followingId: 1 }                  — who follows this user?
```

### Collection: `likes`
```js
{
  _id: ObjectId,
  userId: ObjectId,           // ref: users
  postId: ObjectId,           // ref: posts
  createdAt: Date
}

// Indexes:
//   { userId: 1, postId: 1 }  unique
//   { postId: 1 }              — like counts / who liked a post
```

### Collection: `notifications`
```js
{
  _id: ObjectId,
  recipientId: ObjectId,      // ref: users
  actorId: ObjectId,          // ref: users
  type: String,               // "like" | "reply" | "follow" | "repost"
  postId: ObjectId,           // ref: posts — nullable
  read: Boolean,              // default: false
  createdAt: Date
}

// Indexes:
//   { recipientId: 1, read: 1, createdAt: -1 }
```

### Collection: `reports`
```js
{
  _id: ObjectId,
  reporterId: ObjectId,       // ref: users
  targetType: String,         // "post" | "user"
  targetId: ObjectId,         // polymorphic ref
  reason: String,             // "spam" | "harassment" | "csam" | "misinformation" | "other"
  details: String,            // nullable, max 500 chars
  status: String,             // "pending" | "actioned" | "dismissed"
  reviewedBy: ObjectId,       // ref: users — nullable
  reviewedAt: Date,
  createdAt: Date
}

// Indexes:
//   { status: 1, createdAt: 1 }  — moderation queue
```

### Collection: `auditLog`
```js
{
  _id: ObjectId,
  actorId: ObjectId,          // ref: users (admin/mod)
  action: String,             // e.g. "user.suspend", "post.hide", "user.delete"
  targetType: String,
  targetId: ObjectId,
  metadata: Object,           // arbitrary context (BSON document)
  createdAt: Date
}

// Indexes:
//   { actorId: 1, createdAt: -1 }
//   { targetId: 1 }
//   { createdAt: -1 }
// Note: no delete permission on this collection for any role
```

### Collection: `sessions`

> **Not used.** Firebase ID tokens are verified statelessly. No server-side session storage required.

### Collection: `featureFlags`
```js
{
  _id: String,                // key IS the _id (e.g. "enable_video_posts")
  enabled: Boolean,
  rolloutPct: Number,         // 0–100
  createdAt: Date,
  updatedAt: Date
}
```

---

## 7. API Design

> REST. Versioned under `/api/v1/`. All responses JSON. Authentication via `Authorization: Bearer <firebase-id-token>` header.

### Auth
```
— Sign-in, registration, email verification, password reset, and
— token refresh are all handled client-side by the Firebase Auth SDK.
— Our API has no login/register endpoints.

POST   /api/v1/auth/complete-profile        — set username on first login (new user provisioning)
POST   /api/v1/auth/link-facebook           — trigger Facebook link flow via Firebase
POST   /api/v1/auth/unlink/:provider        — unlink a provider from the Firebase account
```

### Users
```
GET    /api/v1/users/:username              — public profile
PATCH  /api/v1/users/me                     — update own profile
GET    /api/v1/users/:username/posts        — user's posts (paginated)
GET    /api/v1/users/:username/followers    — follower list
GET    /api/v1/users/:username/following    — following list
POST   /api/v1/users/:username/follow       — follow a user
DELETE /api/v1/users/:username/follow       — unfollow a user
```

### Posts
```
GET    /api/v1/posts/:id                    — single post + metadata
POST   /api/v1/posts                        — create post
DELETE /api/v1/posts/:id                    — delete own post
POST   /api/v1/posts/:id/like               — like
DELETE /api/v1/posts/:id/like               — unlike
POST   /api/v1/posts/:id/repost             — repost
DELETE /api/v1/posts/:id/repost             — undo repost
GET    /api/v1/posts/:id/replies            — reply thread (paginated)
```

### Feed
```
GET    /api/v1/feed                         — following feed (paginated, cursor-based)
GET    /api/v1/feed/public                  — global public timeline
```

### Media
```
POST   /api/v1/media/upload-url             — request pre-signed upload URL
POST   /api/v1/media/:id/complete           — notify upload complete, enqueue processing
GET    /api/v1/media/:id                    — get media status + CDN URL
```

### Notifications
```
GET    /api/v1/notifications                — list notifications (paginated)
PATCH  /api/v1/notifications/:id/read       — mark one read
PATCH  /api/v1/notifications/read-all       — mark all read
```

### Push Notification Device Tokens
```
POST   /api/v1/me/fcm-token                 — register FCM device token for current user
DELETE /api/v1/me/fcm-token                 — unregister FCM token (on logout / permission revoked)
```

### Search
```
GET    /api/v1/search/users?q=              — search users by username / display name
GET    /api/v1/search/posts?q=              — full-text search over posts
```

### Reports
```
POST   /api/v1/reports                      — submit a report
```

### Health
```
GET    /healthz                              — liveness
GET    /readyz                               — readiness
```

### Admin (requires admin or moderator role)
```
GET    /api/v1/admin/users                   — list/search users
GET    /api/v1/admin/users/:id               — user detail with admin metadata
PATCH  /api/v1/admin/users/:id/suspend       — suspend account
PATCH  /api/v1/admin/users/:id/unsuspend     — unsuspend account
DELETE /api/v1/admin/users/:id               — delete account
PATCH  /api/v1/admin/users/:id/role          — change role (admin only)

GET    /api/v1/admin/posts/:id               — post detail with admin metadata
PATCH  /api/v1/admin/posts/:id/hide          — hide post
PATCH  /api/v1/admin/posts/:id/unhide        — unhide post
DELETE /api/v1/admin/posts/:id               — delete any post

GET    /api/v1/admin/reports                 — moderation queue
PATCH  /api/v1/admin/reports/:id             — action or dismiss report

GET    /api/v1/admin/audit-log               — query audit log

GET    /api/v1/admin/flags                   — list feature flags
PATCH  /api/v1/admin/flags/:key              — update feature flag
```

### Pagination

All list endpoints use **cursor-based pagination** (not offset):
```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6IjEyMyJ9",
    "hasMore": true
  }
}
```
Cursor is an opaque base64-encoded value. Clients pass it as `?cursor=` on the next request.

---

## 8. Authentication & Authorization

### Identity Provider: Firebase Auth

All authentication is handled by **Firebase Auth** — we do not implement our own credential storage, OAuth flows, password hashing, or token issuance. Firebase handles all of that. Our ASP.NET Core backend validates Firebase ID tokens on every request.

**Auth flow:**
```
1. Client authenticates via Firebase Auth SDK (Ionic/Angular app)
   - Email/password sign-in
   - Google Sign-In (OAuth via Firebase)
   - Facebook Sign-In (OAuth via Firebase)

2. Firebase issues a short-lived ID token (JWT, 1 hour TTL)

3. Client sends ID token in Authorization header:
   Authorization: Bearer <firebase-id-token>

4. ASP.NET Core middleware verifies token against Firebase public keys
   - Validates signature, expiry, audience, issuer
   - Extracts uid (Firebase user ID) and claims

5. Middleware looks up or creates a User document in MongoDB
   using the Firebase uid as the stable identifier

6. Request proceeds with authenticated user context
```

**Token refresh:** The Firebase client SDK handles ID token refresh automatically (tokens expire every hour). No refresh endpoint needed on our API.

**No server-side session storage.** Firebase ID tokens are verified statelessly using Firebase's public JWKS endpoint. Redis is not needed for session management as a result — Redis usage is for caching only (see §12).

### Supported Auth Methods

| Method | Provider | Notes |
|---|---|---|
| Email & Password | Firebase Auth | Firebase handles hashing, email verification, password reset |
| Google Sign-In | Google (via Firebase) | Standard OAuth, managed by Firebase |
| Facebook Sign-In | Meta (via Firebase) | Requires Meta app registration |

> **Apple Sign-In is not supported.** Required for iOS App Store submission but not in scope — no Apple Developer account. Android and web-only for now.

### Firebase User ↔ MongoDB User Mapping

- `firebase_uid` is stored on the `users` document as the stable link between Firebase and our DB
- On first successful token verification for a new `firebase_uid`, a `users` document is created (provisioned from Firebase profile: email, display name, photo URL)
- Username is derived from the display name and made unique; user is prompted to confirm/change on first login
- Multiple providers can be linked to the same Firebase account (managed in Firebase, not our DB)

### Authorization Model

| Action | Rule |
|---|---|
| Read posts / profiles | Public by default (private accounts post-MVP) |
| Create post | Authenticated users only |
| Delete post | Author only |
| Edit profile | Account owner only |
| Follow/unfollow | Authenticated users only |
| Like / repost | Authenticated users only |
| Admin routes | Admin role required |
| Moderation actions | Moderator or Admin role required |

Roles are stored in our MongoDB `users` document. The Firebase ID token carries the `firebase_uid`; our middleware resolves the role from MongoDB.

### Roles

```
user       — standard authenticated user (default)
moderator  — can hide posts, suspend users
admin      — full access including admin dashboard
```

### Security Controls
- All API endpoints require a valid Firebase ID token (except public read endpoints)
- Token verification uses Firebase Admin SDK for .NET — validates against Firebase JWKS, checks `aud` and `iss` claims
- Rate limiting enforced at Zuplo (not the application layer)
- Account suspension: suspended users receive a 403 on every request — checked after token verification
- Account lockout, brute-force protection, and suspicious login detection are handled by Firebase Auth
- Firebase credentials and service account keys never committed to source control; stored in secrets manager

## 9. Media Processing Pipeline

User-generated media (avatars, post images, post videos) must be processed asynchronously. Blocking the request cycle on image transcoding or video encoding is not acceptable.

### Supported Media Types

| Type | Where Used | Formats Accepted |
|---|---|---|
| Avatar image | User profile | JPEG, PNG, WebP, GIF (static only) |
| Post image | Post attachments (up to 4) | JPEG, PNG, WebP, GIF (animated OK) |
| Post video | Post attachments (1 per post) | MP4, MOV, WebM |

### Upload Flow

Image and video uploads follow different paths because they are handled by different services.

Both images and videos use the same S3-compatible presigned URL pattern against Cloudflare R2.

**Images (avatars, post photos):**
```
1. Client requests a presigned R2 upload URL from API
   POST /media/upload-url  →  { uploadUrl, mediaId, r2Key }
   (API generates S3 presigned PUT URL against R2 using AWS SDK)

2. Client uploads image directly to R2
   PUT {uploadUrl}  (bypasses API server — R2 receives the file directly)

3. Client notifies API the upload is complete
   POST /media/:mediaId/complete

4. API marks media document status = ready; no worker processing required
   Cloudflare Image Resizing handles transforms on-demand at delivery time via URL params

5. Image delivered via Cloudflare CDN:
   https://assets.coffeepotsocial.com/{r2Key}?width=400&height=400&fit=cover&format=webp
   (Cloudflare Image Resizing intercepts the CDN request and transforms on the fly)
```

**Videos (post video attachments — post-MVP):**
```
1. Client requests a presigned R2 upload URL from API
   POST /media/upload-url  →  { uploadUrl, mediaId, r2Key }

2. Client uploads video directly to R2
   PUT {uploadUrl}  (bypasses API server)

3. Client notifies API the upload is complete
   POST /media/:mediaId/complete

4. API triggers in-process VideoProcessingWorker (IHostedService)

5. Worker transcodes video (FFmpeg), stores outputs back to R2, updates Media document

6. Client polls or receives SSE event when media status = ready

7. Post references the mediaId once status = ready
   Post-MVP upgrade path: Cloudflare Stream for adaptive bitrate (HLS) without
   self-managed FFmpeg
```

### Image Processing (Cloudflare Image Resizing)

Images are **not processed in-process**. After upload to R2, Cloudflare Image Resizing handles all transforms on-demand at CDN delivery time — no worker, no ImageSharp, no pre-generating variants.

- **Validation (pre-upload):** verify file magic bytes on the server before issuing the presigned URL; reject if not JPEG/PNG/WebP/GIF; enforce 20MB max
- **Safety scan:** optional perceptual hash check against known CSAM databases before issuing the upload URL
- **EXIF strip:** strip EXIF metadata server-side before upload (or as a lightweight pre-process step); R2 does not strip automatically unlike Cloudflare Images
- **Resizing & format:** handled at CDN delivery time via URL query parameters:
  - Avatar large: `?width=400&height=400&fit=cover&format=webp`
  - Avatar small: `?width=100&height=100&fit=cover&format=webp`
  - Post image: `?width=1200&fit=scale-down&format=webp`
  - Thumbnail: `?width=640&height=360&fit=cover&format=webp`
  - `format=auto` can be used to let Cloudflare serve WebP or AVIF based on browser `Accept` header
- **Animated GIF:** served as-is from R2 via CDN; convert to WebP animation as a pre-upload step if size reduction is needed
- **Storage:** R2 holds the original upload; Cloudflare Image Resizing transforms are computed and cached at the edge

### Video Processing (Worker)

Video processing is the most resource-intensive pipeline stage:

- **Validation:** max file size 512MB; max duration 3 minutes (MVP)
- **Safety scan:** same as images
- **Transcoding (parallel):**
  - 1080p MP4 (H.264, AAC) — high quality
  - 720p MP4 — default delivery
  - 360p MP4 — low-bandwidth fallback
- **Thumbnail extraction:** frame at 1s, resized to 640×360
- **HLS packaging (post-MVP):** adaptive bitrate streaming for large videos
- **Store all outputs** to object storage; CDN distributes

### Worker Architecture

For MVP, media processing and background tasks run as **in-process `IHostedService` background workers** within the ASP.NET Core application — no separate queue infrastructure required.

```
┌──────────────────────────────────────────────────┐
│            ASP.NET Core Application              │
│                                                  │
│  ┌─────────────────────────────────────────┐    │
│  │   Background Services (IHostedService)  │    │
│  │  - VideoProcessingWorker (FFmpeg)       │    │
│  │  - NotificationWorker                   │    │
│  │  - MediaCleanupWorker                   │    │
│  │  - NewsletterSyncWorker (scheduled)     │    │
│  └──────────────┬──────────────────────────┘    │
└─────────────────┼────────────────────────────────┘
                  │
     ┌────────────┴─────────────┐
     │                         │
  MongoDB                  Cloudflare R2
  (video job status)       (video output)
```

Photo/avatar transforms are handled by Cloudflare Image Resizing at CDN delivery time — no worker involved.

**Post-MVP scaling path:** If media processing volume or latency becomes a bottleneck, extract workers to separate containers and introduce **DigitalOcean Managed Kafka** as the queue backbone. The `IHostedService` abstraction makes this migration straightforward — swap the in-memory channel for a Kafka consumer.

- FFmpeg via process invocation for video transcoding (images handled entirely by Cloudflare Image Resizing at the CDN layer — no in-process library)
- Job state tracked as `status` field on the `media` MongoDB document (no separate queue collection needed for MVP)
- Failed jobs: retry logic within the background service (max 3 attempts with backoff); marked `failed` in DB on exhaustion

### Media Lifecycle

- Unattached media (upload complete but no post/avatar reference after 24h) → deleted by a cleanup job
- Deleted posts → media marked for deletion; hard-deleted after 7-day grace period
- Users can request account deletion → all media purged within 30 days (GDPR/privacy compliance)

---

## 9a. Push Notifications (FCM)

Push notifications are delivered via **Firebase Cloud Messaging (FCM)** to Android and web (PWA) clients.

### How It Works

```
1. On login (or permission granted), Ionic app requests FCM token from Firebase SDK
2. App sends token to our API: POST /api/v1/me/fcm-token
3. Token stored in user's fcmTokens[] array in MongoDB
4. When a notification event occurs (like, reply, follow, repost):
   - API creates a Notification document in MongoDB (in-app feed)
   - ASP.NET Core IHostedService NotificationWorker sends FCM push message
     via Firebase Admin SDK for .NET
5. FCM delivers push to user's device(s)
```

### Token Management
- Users may be logged in on multiple devices — `fcmTokens` is an array
- Tokens are removed on logout (`DELETE /api/v1/me/fcm-token`)
- Stale/invalid tokens (FCM returns `UNREGISTERED` error) are pruned automatically
- When a user is suspended, notifications are not sent

### Notification Types Triggering Push

| Event | Push? |
|---|---|
| Someone liked your post | Yes |
| Someone replied to your post | Yes |
| Someone followed you | Yes |
| Someone reposted your post | Yes |

### Web Push
- PWA supports push notifications via FCM Web SDK (requires HTTPS and notification permission)
- Ionic Capacitor provides native Android push via FCM automatically

---

---

## 9c. Newsletter Opt-In & Sync

On signup, users are offered the option to join the CoffeePotSocial newsletter. The checkbox is **pre-checked** — users must actively uncheck to opt out.

### Signup UI
- Checkbox label: *"Also add me to the Coffee Pot Newsletter"*
- Pre-checked: `true`
- Value posted to our API along with the rest of the registration payload and stored as `newsletterOptIn: true/false` on the `users` document
- Only shown on email/password and social sign-up flows (not on subsequent logins)

### Sync Flow

```
1. User signs up with newsletterOptIn = true → stored in MongoDB, newsletterSynced = false

2. NewsletterSyncWorker (IHostedService, scheduled — e.g. every 15 minutes or nightly)
   queries: { newsletterOptIn: true, newsletterSynced: false }

3. For each matched user, worker calls the newsletter provider API
   to add the email address (and optionally displayName) as a subscriber

4. On success, worker marks newsletterSynced = true on the user document
   On failure, leaves newsletterSynced = false — will be retried on next run

5. If a user later opts out (account settings toggle), newsletterOptIn is set to false
   A separate unsubscribe call is made to the newsletter provider API
```

### Notes
- Newsletter provider is **Beehiiv** — sync uses the Beehiiv REST API (`POST /v2/publications/{id}/subscriptions`)
- `newsletterSynced: true` is set once Beehiiv confirms the subscription (HTTP 200/201)
- The `newsletterSynced` flag makes the sync idempotent — safe to re-run without double-subscribing
- User emails are never shared externally beyond the newsletter provider
- Opt-out must be available in account settings (GDPR requirement for pre-checked consent)
- The sync worker is lightweight and low-frequency; no dedicated infrastructure needed

---

## 9b. Analytics (Google Analytics)

Google Analytics is integrated client-side in the Ionic app via the **Firebase SDK** (GA is part of the Firebase suite).

### What We Track
- Screen views / page navigation
- Key events: `sign_up`, `login`, `post_created`, `post_liked`, `user_followed`
- Retention and engagement metrics (DAU/MAU)
- Traffic sources for web version

### What We Do Not Track
- Post content
- Private user data
- Any data that would conflict with GDPR/privacy obligations

### Implementation Notes
- Analytics initialised once in the Ionic Angular app module via `AngularFireAnalytics` (or `firebase/analytics`)
- Event calls are fire-and-forget — never block user interactions
- Analytics is **disabled in local/development environment** to avoid polluting production data

---

---

## 10. Monitoring & Observability

A production system that isn't observable is a system you can't operate. Observability is not optional.

### The Three Pillars

#### 1. Structured Logging
- All application logs emitted as **structured JSON** (not plaintext)
- Log levels: `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`
- Every request logged with: `requestId`, `userId` (if authenticated), `method`, `path`, `statusCode`, `durationMs`
- Sentry captures errors and exceptions with full stack traces and request context automatically
- Logs aggregator TBD (DigitalOcean built-in log forwarding likely sufficient for MVP)
- **Never log:** passwords, tokens, full OAuth credentials, PII beyond userId

#### 2. Metrics
Collected via Prometheus-compatible instrumentation and visualised in Grafana:

| Metric | Type | Description |
|---|---|---|
| `http_request_duration_seconds` | Histogram | API response time by route |
| `http_requests_total` | Counter | Request count by method/path/status |
| `db_query_duration_seconds` | Histogram | Database query latency |
| `cache_hit_ratio` | Gauge | Redis cache hit/miss rate |
| `background_worker_active` | Gauge | In-process background workers currently running |
| `background_worker_failures_total` | Counter | Failed background processing jobs by type |
| `active_users_gauge` | Gauge | Currently active sessions |
| `posts_created_total` | Counter | Post creation rate |
| `media_upload_bytes_total` | Counter | Total media ingested |

#### 3. Distributed Tracing
- Request tracing via OpenTelemetry (OTEL) — trace ID propagated across API → queue → worker
- Traces exported to **Sentry** (Sentry supports OTEL-compatible trace ingestion)
- Enables root-cause analysis for slow or failed requests that cross service boundaries

### Alerting

| Alert | Condition | Severity |
|---|---|---|
| High error rate | 5xx rate > 1% over 5 min | Critical |
| API latency degraded | p95 > 1s over 5 min | Warning |
| DB connection pool exhausted | Pool utilisation > 90% | Critical |
| Queue depth spiking | Any queue > 1000 pending jobs | Warning | *(post-MVP — if Kafka is adopted)* |
| Worker DLQ non-empty | Dead-letter queue depth > 0 | Warning | *(post-MVP — if Kafka is adopted)* |
| Disk usage high | Object storage > 80% capacity | Warning |
| Failed login spike | > 100 failed logins/min | Warning |

Alerts route to: email (all), PagerDuty/on-call rotation (Critical only).

### Health Checks

```
GET /healthz          — liveness probe (is the process alive?)
GET /readyz            — readiness probe (is the app ready to serve traffic?)
```

`/readyz` checks: DB connection OK, Redis connection OK, queue connection OK. Used by load balancer and container orchestration to route/restart unhealthy instances.

### Uptime Monitoring

- External uptime checks from at least 2 geographic regions (e.g. BetterStack, UptimeRobot)
- Public status page (e.g. [status.coffeepotsocial.com](https://status.coffeepotsocial.com)) — automated updates on incidents
- SLA targets tracked per calendar month

---

## 11. Admin Tooling

An internal admin dashboard is a requirement — not a nice-to-have. Operating a social platform without moderation and management tools is how things go wrong fast.

### Admin Dashboard

The admin dashboard is a **separate Angular standalone app** located in the `admin/` directory. It is deployed independently from the main `clientapp/` and is accessible only to `admin` and `moderator` roles. It communicates with the same `api/` backend, using the same Firebase Auth tokens, but targets the `/api/v1/admin/*` route namespace.

#### User Management
- Search users by username, email, ID
- View user profile, post history, follower graph
- Suspend / unsuspend account (immediate session revocation)
- Permanently delete account and all associated data
- Manually verify email address
- Reset user password (generates one-time link)
- Promote/demote roles (admin only)
- View login history and active sessions per user

#### Content Moderation
- View reported posts (user-submitted reports)
- Hide / unhide a post (post hidden from all feeds; visible to author with notice)
- Delete any post with audit log entry
- View post with all metadata: author, timestamps, media, report count
- Bulk actions: hide/delete multiple posts
- Moderator action log (who did what, when)

#### Platform Health
- Dashboard view of key metrics (pulled from monitoring stack):
  - Active users (DAU/MAU)
  - Posts per hour
  - New registrations per day
  - Error rate
  - Queue depth
- Link-out to Grafana for deep metrics

#### Feature Flags
- Simple boolean feature flag system
- Flags controllable from admin UI without deployment
- Use cases: roll out features gradually, kill-switch for broken features, A/B experiments
- Flags available to: all users, % of users (percentage rollout), specific user IDs

#### Audit Log
- Immutable append-only log of all admin/moderator actions
- Fields: `timestamp`, `actorId`, `actorRole`, `action`, `targetType`, `targetId`, `metadata`
- Queryable by actor, action type, date range
- Not deletable by any role — integrity preserved

### Reporting (User-Facing)

Users can report posts and accounts:

```
POST /reports
{
  "targetType": "post" | "user",
  "targetId": "...",
  "reason": "spam" | "harassment" | "csam" | "misinformation" | "other",
  "details": "..." (optional, max 500 chars)
}
```

Reports appear in the admin moderation queue.

---

## 12. Scalability & Infrastructure

### Stateless API Design

API servers are stateless — all shared state lives in Redis or the database. This means:
- Any API instance can serve any request
- Instances can be added or removed without downtime
- Load balancer can perform health checks and route around failed instances

### Database Scalability Path

| Stage | Strategy |
|---|---|
| MVP | DigitalOcean managed MongoDB cluster (3-node replica set — built in) |
| Growth | Add read replicas; route analytics/search queries to secondary nodes |
| Scale | MongoDB Atlas-compatible sharding if needed; consider Atlas migration for global clusters |

No premature optimisation — the DO managed cluster handles replication, failover, and automated backups from day one. The document model also means we avoid expensive JOINs for social graph queries.

**Indexes defined in data models (§6) are mandatory from day one.** MongoDB without indexes on high-cardinality queries will full-scan collections at scale.

### Caching Strategy (Redis)

| Cached Data | TTL | Invalidation |
|---|---|---|
| User session | 7 days (rolling) | Explicit logout / admin revoke |
| Public timeline feed | 30 seconds | TTL expiry |
| User profile (public) | 5 minutes | On profile update |
| Post data | 5 minutes | On post delete |
| Rate limit counters | Per window (60s) | TTL expiry |
| Feature flags | 60 seconds | On update |

### Queue & Worker Scaling

- Workers are horizontally scalable — add more worker containers to increase throughput
- Video processing workers are I/O and CPU heavy — sized larger (more RAM/CPU) than other workers
- Auto-scaling based on queue depth (if running on a platform that supports it)

### Infrastructure

| Component | Approach |
|---|---|
| **Containerisation** | Docker for all services |
| **Orchestration** | Docker Compose for local dev; DigitalOcean App Platform or Kubernetes for production (TBD) |
| **IaC** | Terraform — all cloud infrastructure (DO resources, DNS, Cloudflare) defined as code from day one |
| **Environments** | `local` → `staging` → `production` — each environment has its own Terraform workspace |
| **CI/CD** | GitHub Actions: lint → test → build → deploy to staging on PR merge; promote to production manually |
| **Database** | DigitalOcean managed MongoDB cluster (handles replication, backups, failover) |
| **Secrets management** | Environment variables via `.env` locally; DigitalOcean App Platform secrets or DO Vault in production; Terraform state stored in remote backend (Cloudflare R2) with encryption |
| **Backups** | Automated daily snapshots via DO managed DB; tested restore procedure; retention: 30 days |

### Local Development

Every developer should be able to run the full stack locally with a single command:

```bash
docker compose up
```

Services brought up locally:
- `api/` — ASP.NET Core API (with hot reload via `dotnet watch`)
- `clientapp/` — Ionic Angular dev server (with hot reload)
- `admin/` — Angular standalone dev server (with hot reload)
- MongoDB (via official Docker image)
- Redis
- MinIO (S3-compatible local blob storage, mirrors Cloudflare R2; works because R2 uses the standard S3 API)
- Mailhog (email capture for transactional email testing)

> **R2 in local dev:** MinIO fully emulates R2's S3-compatible API — presigned URLs, bucket operations, and AWS SDK calls all work identically. Configure the AWS SDK to point at `http://localhost:9000` locally and `https://<account>.r2.cloudflarestorage.com` in production. Cloudflare Image Resizing has no local emulator; during local dev, serve images directly from MinIO without transforms, and test resizing against the real CF CDN in a staging environment.

Seed data script (dotnet run --project tools/seed) to populate local DB with test users, posts, follows.

---

## 13. Security

Security is not a phase — it's a continuous practice. This section tracks specific controls.

### Security Controls
- All user input validated and sanitised at the API boundary (ASP.NET Core model validation + FluentValidation)
- **NoSQL injection prevention:** never construct MongoDB queries from raw user input strings; use the official .NET MongoDB driver's typed builders (`FilterDefinitionBuilder<T>`) exclusively
- HTML sanitisation on any content rendered in the Ionic frontend (prevent XSS — Ionic renders as WebView)
- File upload: type validated by magic bytes, not just extension or MIME header
- Request body size limits enforced at both Zuplo (gateway) and ASP.NET Core levels
- Firebase service account key stored in secrets manager; never committed to source control

### Transport Security
- HTTPS enforced everywhere — HTTP redirects to HTTPS
- HSTS header with `includeSubDomains`
- TLS 1.2 minimum; TLS 1.3 preferred
- Secure + HttpOnly + SameSite cookies

### Rate Limiting
Primary enforcement at **Zuplo** (API gateway layer) — the ASP.NET Core backend is not the first line of defence:

| Endpoint Group | Limit |
|---|---|
| Auth (login, register) | 10 req/min per IP |
| Post creation | 30 posts/hour per user |
| Media upload URL requests | 20/hour per user |
| General API | 300 req/min per authenticated user |
| Public/unauthenticated | 60 req/min per IP |

### Dependency Security
- `dotnet list package --vulnerable` run in CI — build fails on high/critical vulnerabilities
- Dependabot enabled for automated security patch PRs (NuGet + npm for Ionic)
- Base Docker images pinned to specific digest; updated on a schedule

### Privacy, Compliance & Consent

**Termly** is the compliance and governance layer for CoffeePotSocial. It provides:

- **Privacy Policy** \u2014 hosted and maintained by Termly; embedded via a link from both `clientapp/` and `admin/`
- **Cookie Consent Banner** \u2014 Termly widget embedded in `clientapp/` (web/PWA); handles GDPR, CCPA, and regional consent requirements automatically
- **Terms of Service** \u2014 hosted and maintained by Termly
- Termly keeps legal documents current with regulatory changes; no manual legal copy maintenance

Additional privacy practices:
- Passwords: never stored in plaintext, never logged (Firebase Auth handles hashing)
- OAuth tokens: used transiently, not persisted beyond what's needed
- User emails: never exposed publicly via API
- EXIF data stripped from all uploaded images before storage
- GDPR-compatible account deletion: all user data purged within 30 days of request
- Audit log entries for deletion requests

### OWASP Top 10 Checklist

| # | Risk | Mitigation |
|---|---|---|
| A01 | Broken Access Control | Role-based checks on every endpoint; tests covering unauthorized access |
| A02 | Cryptographic Failures | Firebase handles password hashing; HTTPS everywhere; no sensitive data in logs |
| A03 | Injection | MongoDB typed query builders (no raw string queries); input validation; XSS sanitisation |
| A04 | Insecure Design | Threat modelling during design; security requirements in tickets |
| A05 | Security Misconfiguration | Hardened defaults; no debug endpoints in production; secrets not in code |
| A06 | Vulnerable Components | Automated dependency scanning in CI |
| A07 | Auth Failures | Firebase Auth handles brute-force, lockout, credential security; Zuplo rate limits our API; suspended users blocked at middleware |
| A08 | Data Integrity Failures | Signed upload URLs; worker validates files before processing |
| A09 | Logging Failures | Structured logs; security events logged; no sensitive data logged |
| A10 | SSRF | No user-controlled URLs fetched server-side (MVP); allowlist if added later |

---

## 14. Non-Functional Requirements

| Concern | Target |
|---|---|
| API response time (p50) | < 100ms |
| API response time (p95) | < 300ms for feed/timeline |
| Media upload (time to available) | Images < 5s; Videos < 60s (720p) |
| Availability | Best-effort for MVP; 99.9% post-launch |
| Security | OWASP Top 10 compliance (see §13) |
| Test coverage | Unit + integration tests for all core paths; E2E for critical flows |
| Accessibility | WCAG 2.1 AA for frontend |
| Mobile targets | iOS, Android, PWA from single Ionic codebase |
| Browser support | Latest 2 versions of Chrome, Firefox, Safari, Edge; Android WebView (Chrome-based) |
| GDPR / Privacy | Account deletion within 30 days; no unnecessary PII stored |
| Audit logging | All admin/mod actions logged immutably |

---

## 15. Open Questions

### Stack Decisions
- [x] ~~Frontend framework~~ → Ionic + Angular
- [x] ~~Backend language/framework~~ → ASP.NET Core Web API (.NET)
- [x] ~~REST or GraphQL~~ → REST
- [x] ~~Database~~ → MongoDB on DigitalOcean
- [x] ~~DNS / CDN~~ → Cloudflare
- [x] ~~API Gateway~~ → Zuplo
- [x] ~~IaC~~ → Terraform
- [x] ~~Error tracking / monitoring~~ → Sentry
- [x] ~~Caching~~ → Redis (feed cache, rate limit counters — not sessions)
- [x] ~~Auth / identity provider~~ → Firebase Auth (email/password, Google, Facebook)
- [x] ~~Push notifications~~ → Firebase Cloud Messaging (FCM)
- [x] ~~Analytics~~ → Google Analytics (via Firebase)
- [x] ~~Job queue~~ → No queue for MVP; in-process `IHostedService` background workers; DigitalOcean Managed Kafka if needed post-MVP
- [x] ~~JWT (stateless) or server-side sessions~~ → Firebase ID tokens (JWTs) verified statelessly; no session storage needed

### Product Decisions
- [ ] What do we call a post? ("Brew"? "Pour"? just "post"?)
- [ ] Character limit — 280 to match convention, or something distinct?
- [ ] Private accounts in MVP or post-MVP?
- [ ] Quote-post in MVP or post-MVP?
- [ ] Should we support ActivityPub / Fediverse compatibility (long-term goal)?

### Infrastructure Decisions
- [x] ~~Hosting provider~~ → DigitalOcean
- [x] ~~Object storage~~ → Cloudflare R2 (S3-compatible, zero egress, native Cloudflare CDN)
- [x] ~~CDN~~ → Cloudflare
- [x] ~~Compliance & consent~~ → Termly
- [ ] DigitalOcean App Platform vs. Kubernetes vs. raw Droplets for container orchestration? (decision can wait until after MVP is running)
- [x] ~~Monitoring stack~~ → Sentry for errors/tracing; DigitalOcean native monitoring for infra metrics
- [ ] Metrics dashboard: DigitalOcean native sufficient, or add Grafana?
- [x] ~~Email provider~~ → Postmark (transactional email)
- [x] ~~Newsletter provider~~ → Beehiiv

### Operational Decisions
- [ ] On-call rotation policy? (Who gets paged, and when?)
- [ ] Public status page tooling? (Instatus, Betterstack, self-hosted?)
- [ ] Backup testing cadence?

---

*This spec is a living document. Open a PR or issue to propose changes.*
