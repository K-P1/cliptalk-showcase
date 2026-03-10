# Engineering Decisions

This document summarizes key engineering choices behind the ClipTalk backend and the tradeoffs involved.

## 1. FastAPI + Async-First Design

**Decision:** Use FastAPI with async endpoints and async SQLAlchemy sessions.

**Why:**

- Native async support for:
  - PostgreSQL (via async SQLAlchemy)
  - external HTTP calls (TMDb, external ratings)
- First-class Pydantic integration for request/response validation.
- Excellent developer ergonomics and typing support.

**Tradeoffs:**

- Async IO requires discipline (no blocking calls in the hot path).
- Some libraries are sync-only and need to be isolated or avoided.

## 2. Feature Modularization & Service Layer

**Decision:** Organize code by feature (auth, users, movies, tv, reviews, comments, watchlist, history, recommendations) and centralize business logic in `services` modules.

**Why:**

- Makes each domain cohesive and easier to reason about.
- Route handlers stay thin and focused on:
  - dependency injection
  - request/response mapping
- Easier to test business logic in isolation from HTTP.

**Tradeoffs:**

- Slightly more boilerplate (routes ⇄ services ⇄ models).
- Requires discipline to avoid logic creeping back into routes.

## 3. Asynchronous PostgreSQL with SQLAlchemy

**Decision:** Use SQLAlchemy’s async engine and async session factory, with Alembic for migrations.

**Why:**

- Mature and widely adopted ORM with strong typing.
- Async sessions integrate cleanly with FastAPI dependencies.
- Alembic provides a safe path for schema evolution.

**Tradeoffs:**

- Async ORM patterns are more complex than simple sync ORMs.
- Some advanced SQLAlchemy features behave differently in async mode and require care.

## 4. Configuration via Pydantic Settings

**Decision:** Centralize configuration in a Pydantic `BaseSettings` class loaded from environment variables and `.env`.

**Why:**

- Strong typing and validation for critical configuration (database URL, Redis URL, TMDb keys, OAuth client IDs/secrets, JWT settings, etc.).
- Single source of truth for environment-specific values.
- Easy local development via `.env` while keeping secrets out of source control.

**Tradeoffs:**

- Requires discipline across the codebase to always depend on the settings object instead of ad hoc env reads.

## 5. JWT Auth with Refresh Tokens & Blacklist

**Decision:** Issue both access and refresh tokens with unique JTIs (token IDs) and store a Redis-backed blacklist for logout/rotation.

**Why:**

- Short-lived access tokens limit damage if leaked.
- Long-lived refresh tokens improve UX without sacrificing control.
- JTI-based blacklist allows:
  - explicit logout
  - revocation of compromised tokens
- Redis provides low-latency storage for blacklist checks.

**Tradeoffs:**

- Additional complexity:
  - token rotation
  - refresh workflow
  - blacklist storage
- Requires consistent handling of token types (access vs refresh) across endpoints.

## 6. OAuth via Authlib (Google/Facebook)

**Decision:** Integrate OAuth providers with a unified flow and map external identities to local users.

**Why:**

- Reduces friction for sign-up and login.
- Avoids storing external access tokens in the main database (they are used only to retrieve profile info and then discarded or short-lived).

**Tradeoffs:**

- Extra setup and configuration in each environment (client IDs/secrets, callbacks).
- Additional failure modes when providers are down or misconfigured.

## 7. TMDb Client Abstraction with Error Handling

**Decision:** Wrap TMDb integration in a dedicated async client using httpx with:

- request normalization
- timeouts and connect timeouts
- detailed error logging
- normalized paginated responses

**Why:**

- Centralizes external API behavior and error handling.
- Pydantic schemas consume normalized data regardless of the external API’s quirks.
- Easier to mock in tests.

**Tradeoffs:**

- Client layer needs to be maintained as TMDb evolves.
- Some low-level error details are intentionally hidden from end users to keep responses consistent.

## 8. Redis Caching Strategy

**Decision:** Use Redis for:

- caching TMDb-heavy endpoints (e.g., trending, search, movie details)
- short-lived user profile caching in auth dependencies
- JWT blacklist for logout/rotation

**Why:**

- Offload repeated TMDb calls and reduce latency for popular endpoints.
- Reduce DB load for frequently accessed user metadata.
- Provide O(1)-ish token revocation checks.

**Tradeoffs:**

- Cache invalidation and TTL choices must be tuned:
  - trending and top-rated endpoints cached for minutes to hours
  - movie details cached longer (e.g., a day) with safe TTLs
- Application must handle Redis outages gracefully (fall back to direct DB/API).

## 9. Health Checks & Startup Validation

**Decision:** Run health checks for critical dependencies (database, Redis, etc.) on startup and expose a `/health` endpoint.

**Why:**

- Fail fast if the environment is misconfigured or dependencies are down.
- Allow orchestration (e.g., container platform) to monitor and restart unhealthy instances.
- Separate statuses for core dependencies vs non-critical services.

**Tradeoffs:**

- Startup is slightly more complex:
  - failing startup for critical checks
  - logging degradations for non-critical dependencies

## 10. Logging & Observability

**Decision:**

- Initialize structured logging at application start.
- Attach request logging middleware capturing:
  - method, path, status code
  - response time
- Enable SQL query timing and performance logging on the DB engine (in debug modes).

**Why:**

- Traceability across requests.
- Easier to diagnose slow queries or failing external calls.
- Better insight into production behavior.

**Tradeoffs:**

- Logging must be tuned to avoid excessive noise.
- Sensitive data must never be logged (careful masking where necessary).

## 11. Code Quality & Testing

**Decision:**

- Use tooling for linting, formatting, and type checking (e.g., Ruff, Black, mypy).
- Write async tests against a separate test database.
- Mock external integrations (TMDb and rating providers) in tests.

**Why:**

- Keeps the codebase consistent and maintainable.
- Reduces regressions when evolving the schema and business logic.
- Separates slow or flaky external integrations from unit tests.

**Tradeoffs:**

- Higher upfront investment in tooling and test setup.
- Requires devs to respect the tooling pipeline when making changes.

---

Collectively, these decisions aim to make ClipTalk’s backend:

- robust under load,
- maintainable in the long term,
- secure for user data and tokens,
- and pleasant to work with for future contributors.
