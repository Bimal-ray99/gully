# Gully - Proximity-First Social Backend

> Go monorepo · Gin · MongoDB Atlas · Redis · Asynq · gorilla/websocket · Firebase FCM · Resend

![Go](https://img.shields.io/badge/Go-1.23-00ADD8?logo=go)
![MongoDB](https://img.shields.io/badge/MongoDB-Atlas-47A248?logo=mongodb)
![Redis](https://img.shields.io/badge/Redis-7-DC382D?logo=redis)
![License](https://img.shields.io/badge/license-MIT-green)

---

## What This Is

Gully is a production-grade backend for a proximity-first social platform — posts, feeds, and emergencies
are all geo-tagged and surfaced only to users within radius. No follow graph. No algorithmic timeline.
You see what's happening around you.

10 distributed systems patterns, 14 MongoDB collections, 46 REST endpoints,
native WebSocket real-time events with Redis pub/sub fanout, background workers for FCM push
and outbox processing, and a trust score engine for local businesses with GPS-gated reviews.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│               Clients · React Native · WebSocket · HTTP              │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ HTTPS / WSS
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     Nginx — Edge Layer                               │
│          TLS termination · WebSocket upgrade · reverse proxy         │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
          ┌─────────────────┴──────────────────┐
          ▼                                    ▼
┌──────────────────┐                ┌──────────────────────┐
│   cmd/api        │                │   cmd/workers         │
│   Gin REST API   │ ←── Redis ──→  │   Asynq consumers    │
│   + WS Hub       │ ←── Pub/Sub ─→ │   notifications       │
│                  │                │   nearby-alerts       │
│   14 modules     │                │   outbox-poller       │
│   46 endpoints   │                │   cleanup ticker      │
│   Middleware:    │                │   trust-score ticker  │
│   JWT · Rate     │                │   timecapsule ticker  │
│   Limit · Errors │                └──────────────────────┘
└────────┬─────────┘
         │
┌────────▼──────────────────────────────────────────────────┐
│                      Data Layer                            │
│                                                            │
│  MongoDB Atlas (14 collections)     Redis                  │
│  ┌──────────────────────────────┐   ┌──────────────────┐  │
│  │ 2dsphere geo indexes         │   │ Pub/Sub channels │  │
│  │ Sparse TTL (vanishing posts) │   │ Feed cache       │  │
│  │ Compound unique constraints  │   │ Sliding window   │  │
│  │ $geoNear aggregation         │   │ rate limiters    │  │
│  └──────────────────────────────┘   └──────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

---

## Why These Patterns

| Pattern | Problem It Solves | Where |
|---|---|---|
| **Outbox Pattern** | API writes post to DB, crashes before enqueuing notification task. Emergency happens — nobody gets alerted. | `internal/workers/outbox.go` |
| **Emergency Saga** | Emergency gets spammed. 3 steps: create → vote → threshold check → compensate. Step 3 failure leaves post stuck as ACTIVE forever. | `internal/modules/emergency/service.go` |
| **Redis Pub/Sub + WS Fanout** | Post created on API instance A. User's WebSocket is on instance B. No socket.io adapter in gorilla/websocket — events dropped silently in multi-instance deploy. | `cmd/api/main.go` · `internal/socket/hub.go` |
| **Sliding Window Rate Limiter** | User bulk-posts 200 emergencies/hour. Every post fans out FCM pushes to thousands. Redis sorted-set + ZRemRangeByScore pipeline, atomic. | `internal/rdb/ratelimit.go` |
| **Sparse TTL Index** | Vanishing posts and lost & found have expiry. Regular posts don't. A non-sparse TTL index would need `expiresAt` on every document or delete regular posts 30 days after epoch. | `internal/db/db.go` |
| **Haversine Area Subscription** | MongoDB has no concept of a WebSocket subscription. Client joins with radius. Server must decide which socket gets which post without a per-event DB query. | `internal/socket/hub.go` |
| **GPS-Gated Reviews** | Anyone can review a restaurant from their couch. Trust scores mean nothing. Must be within 200m. Proximity weight decays linearly — review from 180m weighs less than from 20m. | `internal/modules/trustscore/service.go` |
| **Distributed Asynq Queue** | FCM multicast to 1000 users is slow. Doing it inline blocks the HTTP response. Push offline to Asynq — API returns 201 immediately, worker fans out async. | `internal/workers/notification.go` |
| **Feed Cache** | Feed is a $geoNear aggregation — full collection scan with distance scoring. Same user opens feed 10 times in 30 seconds. 10× full aggregation for identical result. | `internal/rdb/cache.go` |
| **Two-Step Email Verification** | Anyone can sign up with another person's email. Unverified emails receive alert spam. Token in DB, link in email, verified flag set on click. Token clears on use. | `internal/modules/auth/service.go` · `internal/email/email.go` |

---

## Numbers

| Dimension | Count |
|---|---|
| REST endpoints | 46 |
| WebSocket events (client→server) | 4 |
| WebSocket events (server→client) | 5 |
| MongoDB collections | 14 |
| Redis pub/sub channels | 4 |
| Background workers | 5 |
| Interval tickers | 4 (1m / 5m / 6h / 6h) |
| Asynq task types | 2 |
| Rate limit buckets | 4 |
| Email triggers | 3 |
| Distributed systems patterns | 10 |

---

## Module Overview

| Module | Prefix | Notable Design |
|---|---|---|
| Auth | `/api/v1/auth` | bcrypt, stateless JWT (access 15m + refresh 30d), email verify token 24h TTL |
| Users | `/api/v1/users` | GPS update rate-limited to 1 req/10s, settings (radius, unit, nearbyAlerts) |
| Posts | `/api/v1/posts` | Geo-tagged on create, type: `regular`, `vanishing` (2h TTL), `timecapsule` |
| Feed | `/api/v1/feed` | $geoNear aggregation, emergencies pinned, cursor pagination, 30s Redis cache |
| Emergency | `/api/v1/emergency` | Business accounts blocked, saga: ACTIVE → vote threshold → REGULAR, WS broadcast |
| Notifications | `/api/v1/notifications` | FCM token register/remove per device, upsert on token value |
| Comments | `/api/v1/posts/:id/comments` | Threaded max depth 2, upvote/downvote, unique vote per user per comment |
| Chat | `/api/v1/chat` | 1:1 conversations, cursor-paginated messages, WS `chat:message` push |
| Business | `/api/v1/business` | Business account type gate, analytics (impressions, spend per ad) |
| Ads | `/api/v1/ads` | Geo-targeted (haversine check), daily budget exhaustion → email alert, ≤2 ads per feed |
| Lost & Found | `/api/v1/lostfound` | 7-day TTL, 2km push radius, outbox fanout, resolve endpoint |
| Trust Score | `/api/v1/trust` | GPS-gated reviews (200m max), proximity weight decay, honesty 1.5× weighted, min 5 reviews |
| Time Capsule | `/api/v1/timecapsule` | Future revealAt, hidden from feed until reveal, 1-min worker publishes WS event |
| Pulse | `/api/v1/pulse` | $geoNear + ~111m grid cells, intensity: low / medium / high crowd density |

---

## WebSocket

Native WebSocket at `wss://host/ws`. No socket.io. JSON envelope protocol:

```json
{ "event": "event-name", "data": { } }
```

**Client → Server**

| Event | Payload | Effect |
|---|---|---|
| `auth:identify` | `{ "token": "eyJ..." }` | Associates socket with user — send immediately after connect |
| `join:area` | `{ "lat", "lng", "radius", "unit" }` | Subscribes to real-time posts in radius |
| `leave:area` | `{}` | Unsubscribes |
| `location:update` | `{ "lat", "lng" }` | Updates area subscription center |

**Server → Client**

| Event | Trigger |
|---|---|
| `post:new` | Post created within subscribed area |
| `post:emergency` | Emergency created within area — client renders alert UI |
| `emergency:update` | Vote threshold reached — status / counts updated |
| `post:expired` | Vanishing post TTL hit — client removes from feed |
| `chat:message` | Incoming DM |

Multi-instance fanout: each API instance subscribes to Redis pub/sub channels. `BroadcastToArea` runs haversine check against all local hub clients. No dropped events across instances.

---

## Workers

| Worker | Interval | What It Does |
|---|---|---|
| `outbox` | 2s poll | Picks pending outbox events (status lock: pending→processing), geo-queries nearby users, enqueues Asynq notification tasks |
| `notification` | Asynq task | Fetches FCM tokens for user IDs, sends Firebase multicast push |
| `nearby-alerts` | Asynq task | Finds users with `nearbyAlerts: true` within 50km, sends FCM push + email fallback |
| `cleanup` | 5m | Deletes expired posts (sparse TTL backup), publishes `room:post_expired` per post |
| `trust-score` | 6h | Recomputes trust score for all places in batches of 50 — weighted dimensions, honesty 1.5×, min 5 reviews |
| `timecapsule` | 1m | Reveals timecapsule posts with `revealAt ≤ now`, publishes `room:new_post`, sets `revealed: true` |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Go 1.23 |
| HTTP + Router | Gin 1.10 |
| Database | MongoDB Atlas · mongo-driver v1.17 |
| Cache / Queue | Redis 7 · go-redis/v9 |
| Background Jobs | Asynq v0.24 |
| Auth | JWT · golang-jwt/jwt/v5 |
| Real-time | gorilla/websocket v1.5 |
| Push Notifications | Firebase Admin SDK (FCM) · firebase.google.com/go/v4 |
| Email | Resend · resend-go/v2 |
| Env Loading | godotenv |
| Password Hashing | bcrypt · golang.org/x/crypto |
| Containers | Docker · Docker Compose |
| Reverse Proxy | Nginx |

---

## Quick Start

```bash
git clone https://github.com/Bimal-ray99/gully-backend
cd gully-backend

cp .env.example .env
# Fill in: MONGODB_URI, REDIS_URL, JWT_SECRET, JWT_REFRESH_SECRET,
#          FCM_PROJECT_ID, FCM_CLIENT_EMAIL, FCM_PRIVATE_KEY,
#          RESEND_API_KEY, FROM_EMAIL, APP_URL

# Full stack (MongoDB + Redis + API + Workers + Nginx)
docker compose -f docker/docker-compose.yml up --build

# API:     http://localhost:80
# Health:  http://localhost:80/health

# Or run natively (requires local Mongo + Redis):
go run ./cmd/api      # terminal 1
go run ./cmd/workers  # terminal 2
```

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `MONGODB_URI` | Yes | MongoDB Atlas connection string |
| `REDIS_URL` | Yes | Redis connection URL |
| `JWT_SECRET` | Yes | Access token secret (min 32 chars) |
| `JWT_REFRESH_SECRET` | Yes | Refresh token secret (min 32 chars) |
| `FCM_PROJECT_ID` | No | Firebase project ID (push notifications) |
| `FCM_CLIENT_EMAIL` | No | Firebase service account email |
| `FCM_PRIVATE_KEY` | No | Firebase private key (literal `\n` sequences, no quotes) |
| `RESEND_API_KEY` | No | Resend API key (email) |
| `FROM_EMAIL` | No | Sender address e.g. `Gully <noreply@gully.app>` |
| `APP_URL` | No | Public URL for email verification links |
| `PORT` | No | HTTP port (default 3000) |

FCM and Resend are optional — omit to disable push notifications and email silently.
