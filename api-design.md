# API Design

This document outlines the main API domains and representative endpoints for ClipTalk.  
Exact implementations and some internal-only details are intentionally omitted.

All endpoints are prefixed with `/api` in production, but paths below are shown without the prefix for readability.

## Authentication & Authorization

### Auth Endpoints

- **POST `/auth/register`**
  - Registers a new user with email, username, and password.
  - Returns a JWT access token, a refresh token, and basic user profile data.
- **POST `/auth/login`**
  - Accepts email/username and password.
  - Returns access + refresh tokens on success.
- **POST `/auth/refresh`**
  - Exchanges a valid refresh token for a new access token (and possibly a new refresh token).
  - Enforces token rotation and blacklist checks.
- **POST `/auth/logout`**
  - Revokes the current access/refresh token pair by inserting their JTI into a Redis blacklist.
- **GET `/auth/me`**
  - Returns the authenticated user’s profile and high-level preference state.

### OAuth Flows

- **GET `/auth/oauth/{provider}`**
  - Redirects user to external OAuth provider (e.g., `google`, `facebook`).
- **GET `/auth/oauth/{provider}/callback`**
  - Handles OAuth callback and:
    - links or creates a user account
    - issues local JWT tokens (access + refresh)

## Users & Profiles

- **GET `/users/{id}`**
  - Public profile view: basic user metadata and high-level stats (e.g., review count).
- **PATCH `/users/me`**
  - Update parts of the current user’s profile (name, bio, avatar, etc.).
- **GET `/users/me/preferences`**
  - Returns current user’s preference configuration used for recommendations.
- **PUT `/users/me/preferences`**
  - Overwrites or updates preference fields such as preferred genres, eras, moods, and discovery mode.

## Movies & TV

### Discovery & Listings

- **GET `/movies/trending`**
  - Query parameters: `time_window=day|week`, `page`
  - Returns a paginated list of normalized movie objects sourced from TMDb.
  - Backed by Redis caching for popular combinations.
- **GET `/movies/top-rated`**
  - Query parameters: `page`
  - Returns normalized top-rated movies.
- **GET `/movies/upcoming`**
  - Query parameters: `region`, `page`
  - Returns upcoming movies for a specific region.
- **GET `/movies/search`**
  - Query parameters: `query`, `year`, `page`
  - Uses TMDb search with normalized results and query-level caching.

Analogous endpoints exist under `/tv/*` for TV shows (e.g., trending, top-rated, search, details).

### Details & Related Content

- **GET `/movies/{tmdb_id}`**
  - Returns:
    - movie details from TMDb (title, overview, release date, images, cast)
    - community rating aggregated from user reviews
    - external ratings (IMDb-like, Rotten Tomatoes-like, Metacritic-like) when available
    - a flag indicating if the response came from cache
- **GET `/movies/{tmdb_id}/similar`**
  - Returns TMDb-based recommendations for similar movies.
- **GET `/movies/{tmdb_id}/trailers`**
  - Returns key trailer videos, combining TMDb and internal trailer records when available.

## Discussions: Reviews & Comments

### Reviews

- **GET `/reviews/movie/{tmdb_id}`**
  - Returns a paginated list of user reviews for a given movie.
- **POST `/reviews/movie/{tmdb_id}`**
  - Authenticated endpoint to submit or update a review with:
    - rating (e.g., 1–5 stars)
    - optional text, optional spoiler flag
- **GET `/reviews/movie/{tmdb_id}/stats`**
  - Returns aggregate metrics such as:
    - average rating
    - total review count
    - distribution buckets

### Comments

- **GET `/comments/movie/{tmdb_id}`**
  - Returns a tree or thread-style list of comments for a movie.
- **POST `/comments/movie/{tmdb_id}`**
  - Authenticated endpoint to add a comment, with optional `parent_id` for replies.
- **POST `/comments/{comment_id}/react`**
  - Authenticated endpoint to like/dislike a comment.
  - Enforces one reaction per user per comment.

Similar comment endpoints exist for TV shows.

## Watchlist & History

### Watchlist

- **GET `/watchlist`**
  - Returns the authenticated user’s watchlist across movies and TV.
- **POST `/watchlist`**
  - Adds an item (movie or TV) to the watchlist.
- **DELETE `/watchlist/{entry_id}`**
  - Removes an item from the user’s watchlist.

### History

- **GET `/history`**
  - Returns viewing history with timestamps.
- **POST `/history`**
  - Records that a user has viewed or interacted with a movie/episode.

## Recommendations

- **GET `/recommendations/feed`**
  - Provides a personalized content feed based on:
    - explicit preferences (genres, eras, moods)
    - implicit signals (ratings, reactions, history)
  - Reads from PostgreSQL and may consult cached TMDb data.

## Response Shape (Generic Example)

Most endpoints return a consistent wrapper structure along the lines of:

```json
{
  "status": "success",
  "message": "Movies fetched successfully",
  "data": {
    "items": [
      /* domain-specific payload */
    ],
    "pagination": {
      "page": 1,
      "total_pages": 10,
      "total_items": 200
    }
  }
}
```

The actual JSON may differ per endpoint, but this conveys the general design philosophy:  
**clear status, human-readable message, and a well-typed `data` field.**
