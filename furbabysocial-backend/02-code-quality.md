# FurBaby Social Backend - Code Quality Analysis

**Date:** 2026-04-18

---

## 1. Architecture Assessment

### Strengths
- **Three-layer design** intended: `api/` (FastAPI routers) → `services/` (business logic) → `dao/` (SQLAlchemy access), plus `schemas/` (Pydantic I/O) and `db/models/` (ORM)
- **Generic DAO base** (`dao/base.py`, 87 LOC) provides typed CRUD (`add`, `get`, `get_all`, `update`, `delete`) — a reusable foundation
- **Centralised exception hierarchy** (`exceptions.py`) built on a `DetailedHTTPException` base
- **Settings as typed Pydantic `BaseSettings`** with per-domain classes (`DBConfig`, `JWTConfig`, `S3Config`, …) cached via `@lru_cache`
- **Structured logging** via `appello-logger` and Elastic APM

### Concerns
- **Layering is only partially enforced.** API routers import DAO classes directly (`api/pets.py:7` uses `CategoryDAO`/`BreedDAO`/`PetDAO`; `api/friends.py:4–5`; `api/notification.py:4`; `api/users.py:6`), bypassing `services/` entirely. Meanwhile services occasionally dodge DAO and run raw SQLAlchemy (`services/posts.py` and `services/friendship.py`). The README's TODO line 125 — *"Move services to DAO"* — confirms this is a known incomplete refactor.
- **`FriendDAO` does not inherit from `BaseDAO`** (`dao/friends.py:10`) while `FriendshipDAO` does (same file:48); a comment admits "should be changed to DAO below."
- **God-module risk**: `services/posts.py` (668 LOC) owns post CRUD, comments, likes, Firebase push, Centrifugo publish, and notification insertion. `create_post` and `like_post` each mix three concerns.
- **No dependency injection for external SDKs.** Firebase, Twilio, S3, Centrifugo clients are instantiated inline in utility modules, making tests harder to mock (and indeed tests do monkey-patch at module paths — see `tests/conftest.py`).

---

## 2. Python/Type Quality

### Configuration
- `requires-python = "==3.12.*"` (`pyproject.toml:8`) but `tool.ruff.target-version = "py311"` (line 136) — **version mismatch** that can mask 3.12-only lint fixes.
- Ruff rule set is ambitious: `E, F, I, SIM, ARG, T20, Q, N, ANN, RET, PT, C4, C90`. `ANN002/ANN003` (annotations on `*args`/`**kwargs`) are ignored — reasonable.
- **Pyright mode = `basic`** (pyproject.toml:166) with `reportUnknownMemberType = false`, `reportUnusedImport = false`, `reportMissingTypeStubs = false` — effectively the weakest useful setting. Pyright is configured but not run in CI.

### Annotation coverage
- Service and DAO functions are largely typed, including return types (~12 of 22 functions in `services/users.py` return-typed).
- API endpoints are **less consistently typed** on return — e.g. `api/friends.py:23–99` routes omit `-> …` annotations. Because FastAPI uses `response_model` instead, this is tolerable but inconsistent with the ruff `ANN` selection.
- No explicit `Any` flood; the codebase doesn't lean on `typing.Any` as a shortcut.

---

## 3. Dependency Version Health

| Dependency | Pin | Status |
|------------|-----|--------|
| **Pydantic** | `>=1.10.21,<2` | **EOL** — v1 no longer receives security fixes. Codebase uses `.dict()` throughout (e.g. `services/posts.py:49`, `api/pets.py:98`), which becomes `.model_dump()` in v2. |
| **SQLAlchemy** | `>=1.4.46,<2` | **Legacy** — v2.0 GA for 2+ years; migration is substantial (declarative mapping, query API). |
| **FastAPI** | `>=0.104.1` | Current, but depends on Pydantic 1 bridge via `fastapi-camelcase<2`. |
| **python-jose** | `>=3.3.0` | OK, but `pyjwt` is also installed — two JWT libraries (dependency bloat). |

This is the largest single quality risk in the codebase: both ORM and validation libraries are a major-version behind.

---

## 4. Large / Complex Files

| Module | LOC | Notes |
|--------|-----|-------|
| `services/posts.py` | 668 | 16 async functions; post/comment/like CRUD + Firebase + Centrifugo + notifications intertwined. `create_post` ≈ 72 lines with nested branches per post type. |
| `services/users.py` | 421 | 22 functions (auth, OTP, tokens, devices, cache). Dense but mostly cohesive. |
| `api/users.py` | 398 | 19 endpoints; thin wrappers, but token/OTP block is intricate. |
| `services/friendship.py` | 253 | Graph operations; mixes direct `select()` calls with DAO usage. |

No other module exceeds 260 lines. The rest of the tree is small and focused.

---

## 5. Code Duplication

| Pattern | Locations | Impact |
|---------|-----------|--------|
| Photo upload loop | `api/pets.py:128–139`, `api/posts.py:46–61` (comment in source: "duplicated code from pets") | Medium |
| DAO-instantiate-and-call (`PetDAO(session).method(...)`) | Repeated across every router module | Low (idiomatic) |
| `try/except NoResultFound / IntegrityError → HTTPException` | 10+ times in `api/pets.py:86–139` | Medium — ripe for a shared error mapper |
| Centrifugo publish-to-users fan-out | `services/posts.py:73–113` (3 near-identical branches per post type) | Medium |

---

## 6. Error Handling

- `exceptions.py` defines a clean hierarchy of `DetailedHTTPException` subclasses (`NotAuthenticated`, `PermissionDenied`, etc.).
- No bare `except Exception:` patterns detected in service/DAO modules.
- `main.py:137–138` wraps `login_admin` in a generic `except Exception` — acceptable as a last-resort admin-login guard but swallows the underlying error class.
- `api/users.py:86` uses `if user := await get_user(...) is None:` — operator precedence means this evaluates the walrus against `is None`, not the intended pattern; a bug-prone idiom worth auditing.

---

## 7. Tooling / Formatting

### Pre-commit (`.pre-commit-config.yaml`)
- `ruff` in `--fix-only` mode + `ruff-format`
- `uv export` to regenerate `requirements.txt` from the lock file

### Gaps
- No `pyright` or `mypy` hook.
- No `pytest` hook.
- No dependency-vulnerability hook (e.g. `pip-audit`).
- Pyright is configured but never executed (neither locally nor in CI).

---

## 8. Documentation

- Endpoint and service docstrings exist but are **one-line summaries** (e.g. `services/users.py:54` — "Returning user with access, refresh, firebase and ws tokens"). No parameter/return documentation.
- README covers local setup, Alembic usage (including the Postgres enum workaround), and a TODO list.
- An `ERD.png` is committed.
- No architectural/ADR documentation.

---

## 9. Dead Code / TODOs

Eight TODOs worth tracking in an issue system:
- `main.py:123` — `/token` endpoint marked deprecated but still routed
- `api/pets.py:98` — pydantic `.dict()` refactor
- `api/posts.py:53` — duplicated photo-upload code from pets
- `api/users.py:154` — extract-to-helper
- `services/friendship.py:35, 164` — reject-friend-request and replaced-by-FriendDAO
- `services/users.py:190, 243` — "raise error in try" and "change in model"

---

## 10. Scoring

| Category | Score (1-10) | Notes |
|----------|-------------|-------|
| Architecture | 7 | Conventional layered shape (api/services/dao/schemas/db/models), generic `BaseDAO`, typed settings per concern, clean exception hierarchy, clear domain boundaries. Dragged by an in-progress services→DAO refactor and one 668-LOC module, but the structure is defensible. |
| Type Safety | 6 | Ruff ANN on, but pyright at `basic` and not CI-enforced |
| Code Organization | 6 | Mostly small modules; `services/posts.py` at 668 LOC is the outlier |
| Naming Conventions | 8 | Consistent PEP8, clear module/domain names |
| Duplication | 6 | Photo upload duplicated across post/pet endpoints; publish fan-out repeated |
| Dependency Management | 3 | Pydantic 1 EOL, SQLAlchemy 1 legacy, two JWT libs, no vulnerability scan |
| Error Handling | 7 | Clean exception hierarchy; one walrus bug, one generic admin-login catch |
| Documentation | 4 | Endpoint docstrings present but thin; no ADRs |
| **Overall** | **6/10** | **Structurally sound with execution-level debt; the lead concerns for this repo are security and scalability (see dedicated reports), not code quality.** |
