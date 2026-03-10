# Architecture Overview

This document describes the backend architecture of ClipTalk at a high level.  
Implementation details are intentionally generalized and redacted.

## High-Level System

ClipTalk follows a fairly standard web architecture:

- **Client:** React SPA (Vite + TailwindCSS)
- **API Layer:** FastAPI application exposing REST endpoints under `/api`
- **Database:** PostgreSQL for persistent data
- **Cache & Messaging:** Redis for:
  - TMDb response caching
  - JWT blacklist for logout/rotation
  - short-lived user session cache
- **External Integrations:**
  - TMDb API for core movie/TV metadata
  - Auxiliary rating sources (e.g., IMDb, Rotten Tomatoes, Metacritic equivalents)
  - OAuth providers (Google, Facebook) for social login

## Backend Structure

The backend codebase is organized into feature-oriented modules. Each feature contains its own:

- **routes** (FastAPI routers)
- **services** (business logic)
- **schemas** (Pydantic models)
- **models** (SQLAlchemy ORM models)

Examples of feature packages:

- `auth/` — authentication, JWT, OAuth flows
- `users/` — user accounts, profiles, preferences
- `movies/` — movie discovery, details, ratings aggregation
- `tv/` — TV show discovery and episodes
- `reviews/` — user reviews and rating stats
- `comments/` — nested comments and reactions
- `watchlist/` — user watchlists for movies/TV
- `history/` — viewing history and engagement tracking
- `recommendations/` — recommendation-oriented endpoints
- `core/` — configuration, dependencies, security, health checks, logging
- `utils/` — TMDb client, caching helpers, external ratings, rate limiting

A thin `main` module wires everything together:

- creates the FastAPI app
- configures middleware (CORS, sessions, logging)
- initializes Redis and the TMDb client
- imports all models so Alembic can detect them
- registers routers for each feature module
- exposes a `/health` endpoint for system status

## Request Flow

At a high level, a typical request flows as follows:

1. **Client Request**
   - The React frontend sends a request to `/api/...` with optional `Authorization: Bearer <token>` header.

2. **FastAPI Routing & Middleware**
   - Request passes through:
     - session middleware
     - CORS middleware
     - request logging middleware
   - It is then dispatched to the appropriate router (e.g., movies, auth, comments).

3. **Dependencies & Authentication**
   - Shared dependencies (e.g., `get_db`, `get_current_user`) are injected:
     - An **async DB session** is created via an async session factory and passed into the route/service.
     - `get_current_user`:
       - Validates JWT using shared auth services.
       - Checks Redis-based blacklist for revoked tokens.
       - Optionally pulls cached user info from Redis, falling back to DB if necessary.

4. **Service Layer (Business Logic)**
   - Route handlers remain thin and delegate to a `services` module.
   - Examples:
     - For movies, services may:
       - check Redis for cached TMDb data
       - call the async TMDb client when necessary
       - map external data to Pydantic schemas
       - persist minimal movie records in the database
     - For auth, services:
       - verify credentials
       - hash/verify passwords
       - create JWT access and refresh tokens with JTI
       - record tokens in a blacklist on logout

5. **Data Layer & External APIs**
   - **PostgreSQL**:
     - Async SQLAlchemy ORM handles queries and relationships.
     - Entities like `User`, `Movie`, `TVShow`, `Review`, `Comment` and `Watchlist` are persisted here.
   - **External APIs**:
     - TMDb client (httpx.AsyncClient) is used to fetch remote data.
     - External ratings client(s) augment information where available.

6. **Response**
   - Service layer returns Pydantic models to the route.
   - FastAPI serializes them into JSON.
   - Responses follow a consistent shape (e.g., wrapped status/message/data structure).

## Mermaid: High-Level Architecture

```mermaid
flowchart LR
    A[Frontend (React)] -->|HTTPS /api/*| B[FastAPI Backend]

    subgraph B[ClipTalk Backend]
        B1[Routing & Middleware]
        B2[Auth & Security]
        B3[Feature Services]
        B4[Async DB Layer]
        B5[Redis Cache & Blacklist]
        B6[External API Clients]
    end

    B1 --> B2
    B1 --> B3
    B2 --> B3
    B3 --> B4
    B3 --> B5
    B3 --> B6

    C[(PostgreSQL)] --- B4
    D[(Redis)] --- B5
    E[TMDb API] --- B6
    F[OAuth Providers\n(Google/Facebook)] --- B2
```

## Design Highlights

- **Feature-based modularization**: Each domain (auth, movies, comments, etc.) is encapsulated with its own routes, services, schemas, and models.
- **Service layer**: Route handlers are kept thin and primarily orchestrate dependency injection and response formatting; all non-trivial business logic lives in services.
- **Async everywhere**: From HTTP endpoints to DB and external API calls, the stack is designed around async IO for better concurrency.
- **Startup health checks**: On startup, the app runs health checks for critical dependencies (e.g., Redis, database) and fails fast if they are not available.
- **Resource lifecycle management**: Redis and external HTTP clients (e.g., TMDb client) are initialized at startup and closed gracefully on shutdown.
