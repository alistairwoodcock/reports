# FurBaby Social Backend - Testing Strategy Gap Analysis

**Date:** 2026-04-18

---

## Current State

| Metric | Value |
|--------|-------|
| Test files | 7 |
| Total test LOC | ~469 |
| Active tests | ~13 |
| Commented-out tests | 3 (post CRUD in `test_crud.py`) |
| Test framework | pytest + pytest-asyncio (`asyncio_mode = "auto"`) |
| Coverage tool | **None configured** |
| Coverage threshold | **None** |
| CI test execution | Yes — but only on `dev` and `stage` branches |
| Prod builds run tests | **No** |

**Verdict: infrastructure is in place, but meaningful coverage has not been invested in.** The test suite covers health-check, auth happy-paths, basic user CRUD, one post creation test, and the Alembic migration stairway. Critical domains (friends, notifications, uploads, services marketplace, WebSocket events) are untested.

---

## Test Inventory

| File | LOC | What it tests |
|------|-----|---------------|
| `tests/conftest.py` | 204 | Fixture plumbing: async engine per test, transactional rollback, session event loop with `alembic downgrade base` finalizer, mock Firebase service, in-memory cache, pre-issued JWTs |
| `tests/test_main.py` | 9 | `/healthcheck` returns 200 |
| `tests/test_migration.py` | 80 | Alembic stairway — every revision upgrades then downgrades cleanly |
| `tests/users/test_auth.py` | 72 | 6 tests: login, OTP send, OTP verify, token issuance |
| `tests/users/test_user.py` | 57 | 4 tests: user get / update / delete with auth |
| `tests/posts/test_crud.py` | 48 | 1 active test (post create); 3 CRUD tests commented out |

---

## Fixture / Mocking Architecture

**Good bones:**
- `TestDBConfig` (`settings.py:150`) reads a separate `TEST_DB_*` env prefix; CI injects these.
- Each test gets a fresh `AsyncSession` with transaction rollback (`conftest.py:173–187`).
- `AsyncClient` + `ASGITransport` exercise the FastAPI app without a network stack.
- Session-scoped event loop with an `alembic downgrade base` finalizer keeps the test DB clean.
- `MockFirebaseService` is auto-used (`conftest.py:154`) so Firebase auth / messaging is stubbed.
- Redis is swapped for `SimpleMemoryCache` via `BaseCache._set_cache()`.

**Gaps:**
- **No mocks for Twilio** — any test that exercises OTP send will either hit real SMS API or rely on incidental failure handling.
- **No mocks for S3 / boto3** — photo-upload paths are effectively untestable without an integration sandbox.
- **No mocks for Centrifugo** — every publish call in `services/posts.py` goes unexercised.
- **Fixture seeding is ad hoc** — no factory (factory_boy / pytest-factoryboy) for Users, Pets, Posts; each test hand-rolls data.

---

## Risk Exposure

Untested (or barely tested) critical paths:

### Critical
1. **Post feed composition** — `services/posts.py` (668 LOC) has 16 public functions; only `create_post` has an active test, and that one is in `test_crud.py`. Public/friends/private feed logic is uncovered.
2. **Friends graph** — `services/friendship.py` (253 LOC) is wholly untested: friend request, accept, reject, block, unblock, blacklist.
3. **Notifications** — push / in-app notification creation on post, comment, like, friend request. Untested.
4. **Photo pipeline** — S3 upload, Pillow validation, photo binding on post/pet creation. Untested.
5. **Centrifugo token issuance and publish** — `utils/centrifugo.py` untested; broken WS tokens would silently break real-time on the mobile client.
6. **OTP lockout / rate limiting** — `otp_attempts`/`otp_minutes` config exists but tests don't exercise the lock path.

### High
7. **Token refresh / rotation** — auth tests stop at issuance; they don't exercise refresh, expiry, or cross-device reuse.
8. **Admin panel auth** (`auth_backend.py`) — untested.
9. **Reports** (`api/report.py`, `services/reports.py`) — untested.
10. **Pet and post edit permissions** — authorization checks ("can user X edit post Y?") are not tested; see SEC-07 in the security report.

### Medium
11. **Cache invalidation** — `UserCache` is read-heavy and write-rare; no test proves stale-cache behaviour on phone/profile change.
12. **Pydantic schema validation edge cases** — max lengths, phone parsing, required fields.
13. **Mention / tagging parsers** (if present in posts).

---

## Recommended Testing Strategy

### Phase 1 — Foundation
- Add `pytest-cov` to the dev dependency group.
- Add a `coverage` section to `pyproject.toml` with `fail_under = 60` for starters.
- Add `pytest-factoryboy` factories for `UserModel`, `PetModel`, `PostModel`, `FriendModel`.
- Add mocks for Twilio (`MockTwilioClient`), S3 (`moto` or hand-rolled), and Centrifugo (`MockCentrifugoClient`) as auto-use fixtures in `conftest.py`.
- Uncomment and finish `tests/posts/test_crud.py`.

### Phase 2 — Service coverage
- **services/posts.py**: feed queries (public/friends/private), comment CRUD, like toggle, block-list exclusion, ownership checks on edit/delete.
- **services/friendship.py**: request/accept/reject/block/unblock, blacklist semantics, duplicate-request guards.
- **services/users.py**: OTP flow including the `otp_attempts` lockout, token refresh rotation.
- **services/notifications.py**: creation on the various trigger events.

### Phase 3 — API contract
- Authorization matrix: for each `edit`/`delete` endpoint, prove 403 when the caller is not the owner (see SEC-07).
- Schema validation: 422s for malformed/oversized bodies.
- Auth matrix: 401 on missing/expired/mismatched-type tokens.

### Phase 4 — CI integration
- Run tests on **every branch** and on the prod build leg (currently skipped — `.drone.yml`).
- Add `--cov --cov-report=xml --cov-fail-under=60` to the pytest invocation.
- Add `ruff check` and `pyright` alongside tests; pyright is configured but never executed.
- Consider a nightly integration job that runs against real Twilio / S3 sandboxes.

---

## Testing Infrastructure Gaps

| Gap | Impact |
|-----|--------|
| No coverage tool or threshold | Progress invisible; easy to regress |
| No Twilio / S3 / Centrifugo mocks | Service layer untestable end-to-end |
| No factories | Brittle, hand-rolled test data |
| Post CRUD tests commented out | Regression of the most-used domain goes unnoticed |
| No authorization tests | Object-level permission bugs ship silently |
| CI skips prod test run | Prod merges are uncovered by CI gate |
| Pre-commit has no pytest hook | Tests not required before local commit |

---

## Coverage Targets

| Milestone | Target | Focus |
|-----------|--------|-------|
| Baseline | 40% | services/users.py, services/posts.py, auth endpoints |
| Core | 65% | All service modules + API authz matrix |
| Comprehensive | 80% | Add DAO edge cases, WebSocket token flow, report pipeline |
