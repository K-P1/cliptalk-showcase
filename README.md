# ClipTalk — Social Movie Discovery Platform

> Documentation-only showcase of the ClipTalk backend architecture.  
> The full application source code is private and **not** included in this repository.

## Overview

ClipTalk is a social movie discovery platform that combines editorial feeds, personalized recommendations, and community features such as reviews, comments, and watchlists.

The backend is a production-grade FastAPI service that:

- integrates with The Movie Database (TMDb) API for movie and TV metadata
- stores user-generated data (reviews, comments, reactions, watchlists, history) in PostgreSQL
- uses Redis for caching and token blacklisting
- exposes a clean, modular REST API for a React frontend

This repository is a **public, documentation-only** showcase intended for recruiters and engineers.  
It focuses on architecture, design decisions, and API/domain modeling without exposing proprietary source code or secrets.

## Tech Stack

- **Language:** Python 3 (async-first)
- **Web Framework:** FastAPI (Starlette under the hood)
- **Database:** PostgreSQL
- **ORM:** SQLAlchemy (async) + Alembic for migrations
- **Caching & Blacklist:** Redis
- **External APIs:** TMDb (movies/TV), plus auxiliary rating providers
- **HTTP Client:** httpx (async)
- **Auth & Security:**
  - JWT access & refresh tokens
  - OAuth2 (Google, Facebook) via Authlib
  - Password hashing with bcrypt (via passlib)
  - Token blacklist stored in Redis
- **Configuration:** Pydantic BaseSettings with `.env` support
- **Frontend (not in this repo):** React + Vite + TailwindCSS

## Key Backend Features

- Modular feature packages for:
  - Authentication & OAuth login
  - Users & profile management
  - Movies & TV discovery (trending, top-rated, search, details)
  - Reviews & rating aggregation
  - Nested comments & reactions
  - Watchlists and viewing history
  - Personalized recommendations based on preferences and behavior
- Asynchronous PostgreSQL access with a centralized async session factory
- TMDb integration with normalized responses and defensive error handling
- Redis-backed caching for TMDb-heavy endpoints and user session data
- Health checks and startup validation for critical dependencies
- Structured logging and request-level logging middleware
- Consistent response schemas and validation via Pydantic models

## Documentation

This repo is organized as a documentation-first showcase:

- [Architecture Overview](architecture.md) — high-level system and backend architecture
- [API Design](api-design.md) — main API domains and representative endpoints
- [Database Schema](database-schema.md) — conceptual data model and relationships
- [Engineering Decisions](engineering-decisions.md) — key technical choices and tradeoffs

Additional assets:

- `screenshots/` — UI screenshots (to be added)
- `diagrams/` — architecture and ER diagrams (source files)

## Source Code & Privacy

The **full implementation is private** and intentionally excluded from this repository.

- No database credentials, API keys, or secrets are present.
- No full model or service implementations are included.
- All examples are simplified and sanitized for public viewing.

If you’d like to discuss implementation details or see specific parts of the architecture in more depth, feel free to reach out.
