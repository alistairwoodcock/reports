# FurBaby Social Backend - Scalability & Architecture Review

**Date:** 2026-04-18

---

## Architecture Overview

```
backend/
├── main.py                # FastAPI app, CORS, middleware, APM, startup hooks
├── settings.py            # Pydantic BaseSettings per domain (DB, JWT, S3, …)
├── auth_backend.py        # SQLAdmin auth backend
├── permissions.py         # is_admin / role checks
├── exceptions.py          # DetailedHTTPException hierarchy
├── admin.py               # SQLAdmin view wiring
├── api/                   # FastAPI routers (users, posts, pets, friends, ws, …)
├── services/              # Business logic (posts, users, friendship, …)
├── dao/                   # Generic BaseDAO + per-entity DAOs
├── db/
│   ├── models/            # SQLAlchemy 1.x ORM (users, posts, pets, …)
│   ├── session.py         # AsyncEngine + session factory
│   └── base.py
├── schemas/               # Pydantic 1.x request/response schemas
├── utils/                 # auth, cache, centrifugo, email, firebase, s3, sms, parsing
├── alembic/versions/      # 30 migrations
├── templates/             # Admin + email Jinja templates
└── tests/                 # pytest suite (~13 active tests)
```

**Pattern:** Layered (Router → Service → DAO) with side-effect integrations isolated in `utils/`. The layering intent is correct but inconsistently applied (see Code Quality report).

---

## Scalability Assessment

### 1. Database Layer

**Pool config** (`db/session.py`): `pool_size=10, max_overflow=5, pool_pre_ping=True, pool_use_lifo=True`. Reasonable for moderate traffic per worker; at 8 workers × gunicorn this becomes 120 potential connections — ensure RDS-side sizing matches. `pool_pre_ping` handles stale connections and `pool_use_lifo` keeps a hot subset warm — subtle but correct choices.

**Eager-loading patterns are correct.** Relationships in `db/models/_posts.py:39–43` use `lazy="selectin"` for photos / comments / likes, and `get_public_posts` uses explicit `joinedload(PostModel.author)`. Selectin loading batches the follow-up query into a single `IN (...)` per relationship, so a feed page of 10 posts issues ~7 queries total (flat in N) — not N+1.

**Indexes — verified against production (2026-04-19 read-only query).** The production database is essentially empty (`user`: 5 rows, `user_device`: 2, everything else 0), so missing indexes haven't started hurting — but adding them now is free while tables are empty, and gets expensive later. Live audit via `pg_indexes` + FK coverage check found:

*Indexes that exist (confirmed in prod):*
- `ix_user_phone_number` on `user.phone_number`
- `uq_like_post_id` on `like(post_id, voter_id)` composite
- `uc_friendship` on `friendship(user_a_id, user_b_id)` composite
- `uc_pet_photo` on `pet_photo(pet_id, photo_id)` composite
- `uq_user_device_device_id` on `user_device.device_id`
- Plus a `pk_*` index on every PK

*Redundant:* every table also has a `uq_*_id` index that duplicates the PK index on the same `id` column (`uq_post_id` alongside `pk_post`, etc., across 13 tables). Harmless but wasteful — each duplicate gets updated on every insert and consumes buffer cache. Worth dropping in a cleanup migration.

*Missing — 25 FK columns have no backing index*, plus the feed-ordering columns on `post`. The ones that matter for the app's hot query paths:

| Missing index | Used by |
|---|---|
| `post(type, created_at DESC)` composite | public-feed ordering — highest-leverage index in the list |
| `post.author_id` | author-profile feed, author filters |
| `friendship.user_b_id` | reverse-direction friend queries (composite leads on `user_a_id` only) |
| `comment.post_id` | comment lookups per post |
| `like.voter_id` | "what did this user like" queries (composite leads on `post_id`) |
| `notification(user_id, created_at DESC)` | notification pagination |
| `notification.sender_id`, `notification.post_id` | related-notification lookups |
| `pet.owner_id` | "pets of user X" queries |
| `pet.breed_id`, `pet.category_id` | breed / category filtering |
| `user_device.user_id` | device list per user |
| `user_photo.user_id`, `user_photo.photo_id` | user-photo joins |
| `report.user_id`, `report.snitch_id`, `report.post_id`, `report.comment_id` | report lookups |

**Composite-index lead-column trap.** Because Postgres composite indexes only support leading-column prefixes, the two composites don't cover reverse-direction queries: `friendship.user_b_id` alone isn't served by `uc_friendship(user_a_id, user_b_id)`, and `like.voter_id` alone isn't served by `uq_like_post_id(post_id, voter_id)`. Both need their own indexes.

Highest-leverage fix is the `post(type, created_at DESC)` composite — one index transforms public-feed latency. The friendship `user_b_id` index is the other must-have for the graph.

**Over-fetching via selectin for scalar aggregates.** This is the real efficiency problem — and it's not N+1. The hybrid properties `comments_count`, `likes_count`, `liked_from_users_ids` (`_posts.py:48–88`) return `len(self.comments)` / list comprehensions over already-loaded collections. Because those relationships are `lazy="selectin"`, the **full** comment and like rows get pulled into memory just to render a scalar count. For a feed page of 10 posts where each has 500 comments and 2 000 likes, that's ~25 000 rows transferred to produce per-post counters.

No schema change is needed — the `.expression` versions of those hybrid properties (`_posts.py:52–88`) already contain the correct scalar subquery. Use them in the main `select()` (e.g. `.add_columns(PostModel.comments_count)`) instead of relying on Python-side `len()`, and drop `lazy="selectin"` on `comments` / `likes` where only counts are needed.

**`NOT IN (many ids)` at scale.** `get_public_posts` loads the caller's full blacklist + blocklist into Python (`services/posts.py:545–549`), then filters posts with `author_id.not_in(list_ids)`. For a power user with thousands of combined entries, this becomes `NOT IN (1, 2, …, 2000)` in the query — Postgres handles it but the planner evaluates it per candidate row. Rewrite as a correlated `NOT EXISTS` or `LEFT JOIN … IS NULL` for large lists.

**Pagination: offset-based** across all list endpoints (`api/posts.py:164`, `api/friends.py:77`, etc.). Offset scans discard the first `N` rows as pages get deep. Keyset pagination (`WHERE created_at < :cursor ORDER BY created_at DESC LIMIT :n`) should be adopted for posts and notifications first — it also needs the `created_at` indexes listed above.

**Bulk operations: none.** Photo binding loops single-row inserts (`services/posts.py:56–59`). Notification fan-out inserts one at a time. `insert(...).values([...])` with a list would collapse these.

**Alembic:** 30 migrations with manual enum workarounds documented in the README. `test_migration.py` runs the full up/down stairway on every revision — genuinely good hygiene that many projects skip.

---

### 2. Async Correctness (most serious scalability issue)

The app is async end-to-end for HTTP I/O and asyncpg, but **three production SDKs are called synchronously inside async handlers**:

| SDK | File:Lines | Called from (request path) |
|-----|-----------|----------------------------|
| firebase-admin | `utils/firebase.py:40, 72` — `auth.create_custom_token`, `messaging.send_each_for_multicast` | login (`services/users.py:64`), post/comment/like (`services/posts.py:137, 365, 470`) |
| Twilio | `utils/sms.py:44` — `client.messages.create` | OTP send (`services/users.py:281`) |
| boto3 (S3) | `utils/s3.py:20, 32, 51` — `resource`, `client`, `Object.put/delete` | photo upload (`services/photos.py`), post create/delete |
| passlib bcrypt | `utils/auth.py:17–22` — CPU-bound | admin login |

None are wrapped in `asyncio.to_thread` / `run_in_threadpool`. Every such call blocks the event loop for the duration of that SDK round-trip. At 8 uvicorn workers with one thread each, ~100 concurrent users triggering Firebase push (10–200 ms each) effectively serialises those requests.

**Fix:** wrap each sync call in `await asyncio.to_thread(...)`. For push/SMS, move them off the request path entirely into a background queue.

---

### 3. Caching

**What's cached** (`utils/cache.py`, aiocache over Redis):
- `UserCache` — TTL 3500 s (~1 h), read after login (`services/users.py:96, 263`).
- `PetCategoryCache`, `PetBreedCache` — TTL 21600 s (6 h) on static reference data.

**What's not cached:** posts, feeds, comments, likes, friend lists, notifications — all hit the DB on every request.

**Invalidation:** essentially absent. `UserCache` is only explicitly refreshed on phone change.

**Fallback:** Redis unavailable at startup falls back to `SimpleMemoryCache` (`main.py:72–90`). Per-worker memory means cache coherence disappears across gunicorn workers — a silent correctness footgun rather than a graceful degradation.

**Recommendation:** Add short-TTL caches on read-hot paths (user-scoped feed, pet lists), plus explicit invalidation hooks on writes. Treat Redis as a hard dependency; fail startup rather than fall back to per-worker memory.

---

### 4. Real-time Layer (Centrifugo)

**Publish pattern** (`utils/centrifugo.py:44–87`, `services/posts.py:74–113`): `ws_client.publish_to_users()` enumerates recipients explicitly. Public post creation fetches **every user** via `UserDAO(session).get_all()` and publishes to each user's private channel (`services/posts.py:93–102`).

**Consequences at scale:**
- 100k users × 1 public post = 100k Centrifugo publish calls per post creation.
- O(N) per event, synchronous in the request path.

**Recommendation:** Use a single public topic (`furbaby:{env}:public`) that all clients subscribe to; reserve per-user channels for mentions / direct events. Alternatively, move fan-out to a background worker.

---

### 5. Background / Deferred Work

**None implemented.** No Celery, ARQ, RQ, APScheduler, or FastAPI `BackgroundTasks`. The README lists S3 orphan cleanup as a TODO (line 128). Consequences:
- Photo deletion on post delete blocks the request (`services/posts.py:309`).
- Push notifications are sent synchronously per event.
- No retry on transient Firebase/Twilio failures.
- No scheduled maintenance (stale notification cleanup, refresh-token GC).

**Recommendation:** Introduce an async task runner (ARQ pairs well with asyncio + Redis already in the stack). Move push, email, and S3 cleanup off the request path.

---

### 6. Real Data-Scale Concerns

| Scenario | Behaviour today | Risk |
|----------|----------------|------|
| 10 k friends per user | `get_black_list`/`get_block_list` load all into memory, then `NOT IN` on posts | High — query plan degrades, `IN` list blows up |
| 100 k posts | Offset pagination, subqueries per post for counts | High — P95 latency climbs linearly with page depth |
| 1 M notifications | Unbounded table growth; no archival | Medium — admin UI and pagination slow over time |
| Viral post | Fan-out publishes to every user at event time | High — single request creates N network calls |

---

### 7. Deployment Shape

`gunicorn.conf.py` (3 lines): `workers = multiprocessing.cpu_count()`, `worker_class = uvicorn.workers.UvicornWorker`, `bind = "0.0.0.0:8000"`. Appropriate for async FastAPI. Stateless; no sticky sessions needed.

**Health check:** `GET /healthcheck` returns a static 200 (`main.py:107–109`) — **does not check DB, Redis, Firebase, or Centrifugo**. An ECS task with broken Redis still passes health. Add a deep-check endpoint for liveness vs readiness semantics.

**APM:** Elastic APM auto-instruments via middleware when `elastic.apm_server_url` is set (`main.py:40–52`). Good.

---

### 8. Rate Limiting / Backpressure

`limits>=3.6.0` is declared but unused. Neither request-level nor per-endpoint limits are wired. `otp_attempts`/`otp_minutes` are application-layer counters checked after the SMS has been sent — too late to prevent abuse. Covered under SEC-03 in the security report.

---

### 9. Hard Upstream Dependencies on Critical Paths

| Upstream | In-path | Timeout | Failure mode today |
|----------|---------|---------|--------------------|
| Twilio SMS | Login/OTP | Unset (SDK default ~30 s) | Login stalls on Twilio outage |
| Firebase Messaging | Post/comment/like | Unset | Request hangs on FCM latency |
| S3 | Photo upload/delete | Unset | Upload hangs on S3 blip |
| Centrifugo | All mutating post events | 1 s client timeout (`utils/centrifugo.py:28`), but publish loop is sync | Partial publishes possible |

**Recommendation:** Explicit, short timeouts on every external call; circuit breaker around Twilio and Firebase; move non-essential integrations out of the request path.

---

## Scaling Readiness Score

| Dimension | Score (1–5) | Notes |
|-----------|-------------|-------|
| Architecture Layering | 3 | Sound pattern, incomplete DAO migration |
| Database | 3 | Correct eager-loading + pool config + stairway migration test; missing indexes on hot FK/ordering columns; over-fetching for scalar aggregates; offset pagination |
| Async Correctness | 1 | Three sync SDKs block the event loop |
| Caching | 2 | Minimal coverage, no invalidation |
| Real-time | 2 | Centrifugo fan-out is O(users) per event |
| Background Tasks | 1 | None implemented |
| Deployment | 3 | Gunicorn/Uvicorn + APM; shallow health check |
| Feature Extensibility | 4 | Clean domain modules make new features easy |
| **Overall** | **2.4/5** | **Functional today; will not scale past a few thousand active users without query-tuning and unblocking the event loop** |

## Top 5 Interventions

1. **Wrap sync SDKs in `asyncio.to_thread`** — highest leverage single change.
2. **Add the missing indexes** on `post(type, created_at DESC)`, `post.author_id`, `friendship.user_b_id`, `comment.post_id`, `like.voter_id`, `notification(user_id, created_at DESC)`, `pet.owner_id`, `user_device.user_id`. Do it now while every table is essentially empty — adding them later at scale is a much harder migration. Drop the 13 redundant `uq_*_id` indexes in the same change.
3. **Stop over-fetching for scalar aggregates** — use the existing `.expression` forms of `comments_count` / `likes_count` as SQL subqueries in the main select; drop `lazy="selectin"` on collections only used for counts.
4. **Move push / SMS / S3 cleanup to a background worker** (ARQ + Redis).
5. **Collapse the Centrifugo public-post fan-out** onto a single topic; adopt keyset pagination on posts / notifications.
