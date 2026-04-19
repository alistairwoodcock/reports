# FurBaby Social Backend - Application Codebase Analysis: Executive Summary

**Date:** 2026-04-18
**Project:** FurBaby Social Backend API (FastAPI)
**Version:** 0.1.0
**Repository:** furbabysocial/backend

---

## Overview

The FurBaby Social backend is a Python 3.12 FastAPI application serving the FurBaby Social mobile app. It exposes a REST API plus a WebSocket real-time layer (via Centrifugo), persists to PostgreSQL through SQLAlchemy 1.x + asyncpg, and integrates with Firebase (custom tokens, push messaging, Firestore), Twilio (OTP SMS), AWS S3 (media), Redis (cache via aiocache), and Elastic APM (observability). An SQLAdmin panel provides internal admin tooling.

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Language/Runtime | Python 3.12 (ruff targets py311 — mismatch) |
| Framework | FastAPI 0.104+ |
| ASGI Server | Gunicorn + Uvicorn workers |
| ORM | SQLAlchemy 1.x (`<2` pin) + asyncpg |
| Migrations | Alembic (30 revisions) |
| Validation | Pydantic 1.x (`<2` pin) — EOL |
| Auth | JWT (python-jose) + OTP via Twilio |
| Admin | SQLAdmin |
| Cache | aiocache on Redis (in-memory fallback) |
| Media | AWS S3 (boto3 — sync SDK) |
| Real-time | Centrifugo (pub/sub) |
| Push | firebase-admin (sync SDK) |
| Observability | Elastic APM + appello-logger (Logstash) |
| CI/CD | Drone CI → ECR → AWS ECS (rolling) |
| Environments | Dev / Stage / Prod (multi-account AWS) |

## Codebase Metrics

| Metric | Value |
|--------|-------|
| Python source files (excl. tests/migrations) | ~72 |
| Total project Python files | 102 |
| Estimated application LOC | ~6,041 |
| API route modules | 9 |
| Service modules | 7 |
| DAO modules | 7 |
| Alembic migrations | 30 |
| Test files | 7 (~469 LOC) |
| Active tests | ~13 |
| Test coverage | **Unmeasured (no coverage tooling)** |

## Risk Assessment

| Area | Risk Level | Summary |
|------|-----------|---------|
| Security | **CRITICAL** | Two Critical findings: production RDS is reachable from the public internet (protected only by shared password), and no rate limiting on OTP / login. Five High: wildcard CORS (admin-panel exposure), unauthenticated photo upload, long-lived unrevokable tokens, plaintext OTPs + hard-coded `1111` backdoor for 22 real phone numbers, IDOR on device / photo endpoints. JWT secret also has an insecure default with no startup validation (Medium). |
| Scalability | **HIGH** | Sync SDKs (Firebase, Twilio, boto3) block the async event loop; over-fetching entire comment/like collections just to compute scalar counts; missing indexes on hot FK and ordering columns; offset pagination everywhere; O(users) fan-out on public posts. |
| Tech Debt | **HIGH** | Pydantic 1.x EOL, SQLAlchemy 1.x EOL, partial `services → dao` refactor. |
| Testing | **HIGH** | ~13 tests total; post CRUD tests commented out; no coverage tracking; no mocks for Twilio/S3/Centrifugo. |
| Code Quality | **LOW-MEDIUM** | Structurally sound — conventional layered shape, generic `BaseDAO`, typed settings, clean exception hierarchy. `services/posts.py` at 668 LOC and a mid-flight `services → dao` refactor are the main execution-level gaps. |
| CI/CD | **MEDIUM** | Deploys work, but no lint/type-check/coverage in CI, no prod test run, no deploy approval gate. |
| Observability | **LOW** | Elastic APM + structured logging via appello-logger / Logstash. |

## Top Recommendations (Priority Order)

1. **Make the production database private** — RDS `PubliclyAccessible=false`, tighten the security group to the ECS task security group for app access plus SSM / bastion for humans, move the instance to a private subnet if it isn't already. Today the DB is reachable from anywhere on the internet and protected only by a shared password; any password leak is an immediate full-prod compromise.
2. **Fix the active security holes immediately** — Wire `slowapi` (the `limits` package is already installed) onto `/auth/otp/send` and login, add authentication to both photo upload endpoints, remove the `"1111"` OTP backdoor and its 22 hard-coded phone numbers, and scope CORS to the SQLAdmin browser origin.
3. **Unblock the async event loop** — Wrap boto3, firebase-admin, and Twilio calls in `asyncio.to_thread` / `run_in_threadpool`. Better yet, move Twilio SMS, Centrifugo broadcast, and S3 uploads off the request path entirely (background tasks + presigned-PUT for S3). These three sync SDKs currently sit directly in request-path handlers (login, OTP, post creation, likes, uploads).
4. **Add database indexes and stop over-fetching for scalar aggregates** — Verified against prod (2026-04-19): 25 FK columns are missing indexes, the `post` table has no `(type, created_at)` composite, and `friendship`/`like` composites only cover one direction of their queries. Do this now while the tables are essentially empty — adding indexes at scale is a much harder migration. Also switch the feed queries to use the existing `.expression` forms of `comments_count` / `likes_count` as SQL subqueries, and drop `lazy="selectin"` on collections that are only used for counting.
5. **Plan the Pydantic 2.x / SQLAlchemy 2.x migration** — Both are pinned `<2`. Pydantic 1.x is out of support; widespread `.dict()` usage and declarative 1.x ORM patterns make this a non-trivial but necessary project.
6. **Raise the test floor** — Uncomment and finish the post CRUD tests, add mocks for Twilio/S3/Centrifugo in `conftest.py`, adopt `pytest-cov` with a gating threshold, and run it in CI on all branches (currently only dev/stage).
7. **Fix CI gaps and add a prod approval gate** — Add `ruff check` and `pyright` to `.drone.yml`; gate the prod ECS deploy behind manual approval; add dependency vulnerability scanning alongside the existing ggshield secret scan.
