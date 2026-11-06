# Requirements: music-db REST API

Every requirement found in the two sources below, in one list. Each requirement has an ID so it can be referenced from issues, tests and reviews.

| Source | Tag | What it is |
|---|---|---|
| [API-Design.md](API-Design.md) (this repo) | **D** | The design: the contract for the API. |
| [`api-spring-music`](../api-spring-music) | **I** | The Spring Boot implementation of the design: its code, configuration, tests, README and CLAUDE.md. |

**D+I** means both sources state the requirement. **I** alone means the implementation added it, or changed what the design said. [Design vs. implementation](#24-design-vs-implementation) lists the places where the two disagree.

**Contents**

1. [General API conventions](#1-general-api-conventions)
2. [Authentication and sessions](#2-authentication-and-sessions)
3. [Roles and authorization](#3-roles-and-authorization)
4. [Account (`/me`)](#4-account-me)
5. [Users](#5-users)
6. [Playlists](#6-playlists)
7. [Playlist tracks](#7-playlist-tracks)
8. [Collaborators](#8-collaborators)
9. [Library](#9-library)
10. [Follows](#10-follows)
11. [Catalog](#11-catalog)
12. [Search](#12-search)
13. [Reference data](#13-reference-data)
14. [Catalog administration](#14-catalog-administration)
15. [Account administration](#15-account-administration)
16. [Errors](#16-errors)
17. [Database integration](#17-database-integration)
18. [Security](#18-security)
19. [Configuration](#19-configuration)
20. [Deployment and operations](#20-deployment-and-operations)
21. [API documentation](#21-api-documentation)
22. [Testing](#22-testing)
23. [Technology and code constraints](#23-technology-and-code-constraints)
24. [Design vs. implementation](#24-design-vs-implementation)
25. [Open items](#25-open-items)

---

## 1. General API conventions

| ID | Requirement | Source |
|---|---|---|
| GEN-01 | The base URL is `/api/v1`. | D+I |
| GEN-02 | The API version comes from the path segment. `v1`, `1` and `1.0` all mean version 1, and an unsupported version (for example `/api/v2/...`) answers `400`. | I |
| GEN-03 | Requests and responses are JSON in UTF-8. | D+I |
| GEN-04 | Field names are `snake_case` and match the database column names. | D+I |
| GEN-05 | Timestamps are ISO 8601 in UTC, for example `"2026-11-05T20:43:11Z"`. | D+I |
| GEN-06 | Durations are returned as `duration_ms`, plus a display string `duration` (`"4:20"`) where a view provides one. | D+I |
| GEN-07 | The display duration is `m:ss`, or `h:mm:ss` for an hour or more. Seconds are truncated, as in the `track_detail` view. | I |
| GEN-08 | Users are addressed by `username`, artists and labels by `slug`, and playlists, albums, tracks, songs, people and genres by numeric ID. An album can also be addressed by artist slug plus album slug. | D+I |
| GEN-09 | List endpoints accept `?limit=` (default 20, minimum 1, maximum 100) and `?offset=` (minimum 0). | D+I |
| GEN-10 | Paged responses have the shape `{ "items": [...], "limit": 20, "offset": 0, "total": 193 }`. | D+I |
| GEN-11 | Lists that change often also accept a keyset cursor that stays stable while items are added: `?after_position=` on playlist entries, and `?before=<saved_at>` on the library. | D+I |
| GEN-12 | The library cursor also accepts `before_id`, which breaks ties between items saved at the same instant. | I |
| GEN-13 | `PATCH` changes only the fields that are sent. Sending `null` clears a nullable field. The API can tell a field that was not sent apart from a field sent as `null`. | I |
| GEN-14 | Catalog reads and public-playlist reads work without a token. Everything under `/me`, and every write, needs one. | D+I |
| GEN-15 | Responses are compressed. | I |
| GEN-16 | Every resource and behavior described in API-Design.md is implemented. | I |

## 2. Authentication and sessions

| ID | Requirement | Source |
|---|---|---|
| AUTH-01 | Requests authenticate with `Authorization: Bearer <token>`. | D+I |
| AUTH-02 | The API reads the token's `sub` claim and finds the user through `app_user.auth_subject`. | D+I |
| AUTH-03 | The `public` (music) schema never stores passwords. | D+I |
| AUTH-04 | The design expects tokens from an external login provider (Auth0, Cognito, Keycloak, Supabase…). | D |
| AUTH-05 | The implementation issues its own JWTs: it is the "login provider" of the design. | I |
| AUTH-06 | `POST /auth/login` takes a `username` (or email) and a `password`, and returns `access_token`, `token_type` (`Bearer`), `expires_in`, `refresh_token` and the refresh token's lifetime. | I |
| AUTH-07 | `POST /auth/refresh` exchanges a refresh token for new tokens. The old refresh token stops working. | I |
| AUTH-08 | `POST /auth/logout` ends the session that a refresh token belongs to. An unknown token is ignored, and the answer is `204`. | I |
| AUTH-09 | Token responses carry `Cache-Control: no-store` and `Pragma: no-cache`. | I |
| AUTH-10 | Access tokens are JWTs signed with HMAC-SHA256 and valid for 15 minutes by default. | I |
| AUTH-11 | Access-token claims: `iss`, `aud`, `sub` (the `auth_subject`), `iat`, `nbf`, `exp`, `jti`, `preferred_username` and `role`. `iss` and `aud` are checked on every request. | I |
| AUTH-12 | Authorization never trusts the token's `role` claim. The role is read from the database on every request, so a role change applies at once. | I |
| AUTH-13 | A token whose account was deleted, or whose `sub` is missing or unknown, is rejected with `401`. | I |
| AUTH-14 | Refresh tokens are opaque (32 random bytes, Base64URL), valid for 14 days by default, single-use, and stored only as SHA-256 hashes. | I |
| AUTH-15 | Every refresh token from one login belongs to a family. Presenting a used or revoked token again revokes the whole family (reuse detection), and the reuse is logged as a warning. | I |
| AUTH-16 | An unknown, expired, used or revoked refresh token answers `401 invalid_grant`. | I |
| AUTH-17 | A wrong password and an unknown user get the same answer (`401 invalid_credentials`). The API checks a dummy hash for an unknown user, so the response time doesn't reveal whether the account exists. | I |
| AUTH-18 | An account without a password (such as the seed users) can't log in. | I |
| AUTH-19 | Passwords are hashed with bcrypt through a delegating encoder (an `{id}` prefix), so the algorithm can be upgraded later. Outdated hashes are re-encoded at login. | I |
| AUTH-20 | Passwords chosen through the API need at least 8 characters, can't be blank, and can be at most 72 bytes in UTF-8 (`weak_password` / `password_too_long`, `422`). | I |
| AUTH-21 | `PUT /me/password` changes the caller's password. It needs the current password (`422 wrong_password` when that is wrong), and it signs out all of the caller's sessions. | I |
| AUTH-22 | Refresh tokens that expired more than a day ago are deleted once a day (default cron `0 30 3 * * *`, `app.jwt.cleanup-cron`). | I |
| AUTH-23 | The API is stateless: no HTTP sessions, no CSRF tokens (bearer tokens only, no cookies), no form login, no HTTP Basic. | I |

### Super user

| ID | Requirement | Source |
|---|---|---|
| AUTH-24 | A super user is created at startup from `APP_ADMIN_USERNAME` and `APP_ADMIN_PASSWORD`, with role `admin`. Nothing is created when either is unset. | I |
| AUTH-25 | If an account with that username already exists, it is made an admin. | I |
| AUTH-26 | `APP_ADMIN_PASSWORD` is only applied to an account that has no password, so a password changed through the API survives restarts. | I |
| AUTH-27 | A newly created super user gets `APP_ADMIN_EMAIL` (default `<username>@localhost`) and `APP_ADMIN_DISPLAY_NAME` (default: the username), and is marked verified. | I |
| AUTH-28 | Startup fails if `APP_ADMIN_USERNAME` doesn't match `^[a-z0-9_.]{3,30}$`. | I |
| AUTH-29 | A super-user password shorter than 12 characters logs a warning at startup. | I |
| AUTH-30 | The super user of this deployment is `oscargm` (user 8, role `admin`). db-postgresql-music seeds the account without a password, and the API sets its password from the environment. | I |
| AUTH-31 | The log says whether the super user was created, or already existed and kept its password. | I |

## 3. Roles and authorization

| ID | Requirement | Source |
|---|---|---|
| ROLE-01 | Catalog writes need an admin-style role. The design suggests adding something like `app_user.role` (`listener`, `curator`, `admin`). `is_verified` marks editorial and artist accounts, and is not a permission. | D |
| ROLE-02 | There are three account roles: `listener`, `curator` and `admin`. Each role includes the ones below it (`admin > curator > listener`). | I |
| ROLE-03 | Roles live in the API-owned `auth` schema (`auth.role`, `auth.user_account.role`), not in `public`. | I |
| ROLE-04 | New accounts are `listener`. | I |
| ROLE-05 | `listener`: everything under `/me`, plus playlists, library and follows. | I |
| ROLE-06 | `curator`: also `/admin/**` (catalog writes). | I |
| ROLE-07 | `admin`: also `/admin/users/**` (accounts) and `/actuator/**` beyond health and info. | I |
| ROLE-08 | `GET /me` also returns the caller's `role`. | I |

### Playlist access

| ID | Requirement | Source |
|---|---|---|
| ROLE-09 | `public` playlists can be opened by everyone, and are listed on the owner's profile and in search. | D+I |
| ROLE-10 | `unlisted` playlists can be opened by anyone with the share link, plus the owner and collaborators. They are not listed. | D+I |
| ROLE-11 | `private` playlists can be opened only by the owner and collaborators. They are not listed. | D+I |
| ROLE-12 | View and edit checks use the database functions `playlist_can_view(playlist_id, user_id)` and `playlist_can_edit(playlist_id, user_id)`. | D+I |
| ROLE-13 | A playlist the caller can't view answers `404`, not `403`, so a private playlist doesn't reveal that it exists. | D+I |
| ROLE-14 | `share_token` is a secret. Only the owner ever receives it. | D+I |
| ROLE-15 | The **owner** can do everything: edit details and visibility, manage collaborators, rotate the share link, and delete. | D+I |
| ROLE-16 | An **editor** can add, move and remove tracks, but can't change the playlist's details. | D+I |
| ROLE-17 | A **viewer** can open the playlist, even a private one. | D+I |

## 4. Account (`/me`)

| ID | Requirement | Source |
|---|---|---|
| ACC-01 | `POST /users` (sign-up) creates the caller's profile, with `username`, `display_name`, `email` and an optional `country_code`. | D+I |
| ACC-02 | In the design, sign-up is called once after the first login with the provider, and the token's `sub` becomes `auth_subject`. | D |
| ACC-03 | In the implementation, sign-up needs no token, also takes a `password`, and creates the profile and its credentials in one transaction. | I |
| ACC-04 | Sign-up answers `201 Created` with `Location: /api/v1/users/{username}` and the profile (`user_id`, `username`, `display_name`, `country_code`, `bio`, `avatar_url`, `is_verified`, `follower_count`, `following_count`, `created_at`). | D+I |
| ACC-05 | A username must match `^[a-z0-9_.]{3,30}$`. Otherwise the answer is `422`. | D+I |
| ACC-06 | An unknown country code answers `422`. Country codes are ISO 3166-1 alpha-2 and are stored in upper case. | D+I |
| ACC-07 | A taken username answers `409 username_taken`, and a taken email `409 email_taken`. | D+I |
| ACC-08 | Validation limits: `display_name` at most 80 characters, `email` a valid address of at most 254 characters, `password` 8 to 72 characters. | I |
| ACC-09 | `GET /me` returns the caller's profile, plus `email`, which is only ever shown to its owner. | D+I |
| ACC-10 | `PATCH /me` changes only the fields that are sent: `display_name`, `bio`, `avatar_url`, `country_code`. It answers `200 OK` with the updated profile. | D+I |
| ACC-11 | `updated_at` is set by a database trigger. | D+I |
| ACC-12 | Usernames are permanent, because they appear in URLs. Sending `username` in `PATCH /me` answers `422 username_immutable`. | D+I |
| ACC-13 | `display_name` can't be cleared or blank. `null` clears `bio`, `avatar_url` and `country_code`. | I |
| ACC-14 | Profile limits: `bio` at most 2000 characters, `avatar_url` an http(s) URL of at most 500 characters. | I |
| ACC-15 | `DELETE /me` answers `204` and deletes the account with everything it owns: playlists, collaborations, library and follows (`ON DELETE CASCADE`). | D+I |
| ACC-16 | After `DELETE /me`, triggers correct everyone's follower counts, and tracks the user added to other people's playlists stay, with `added_by = NULL`. | D+I |
| ACC-17 | `DELETE /me` also removes the account's credentials and sessions, so its tokens stop working. | I |
| ACC-18 | `GET /me/playlists` lists the playlists the caller owns or collaborates on, with `my_role` on each, from `playlist_summary` united with `playlist_collaborator`. | D+I |
| ACC-19 | `GET /me/feed` lists the newest entries in playlists the caller follows or collaborates on, ordered by `playlist_track.added_at DESC` (example 23 in `examples.sql`). | D+I |

## 5. Users

| ID | Requirement | Source |
|---|---|---|
| USR-01 | `GET /users/{username}` returns the public profile: `user_id`, `username`, `display_name`, `bio`, `country_code`, `avatar_url`, `is_verified`, `follower_count`, `following_count`. | D+I |
| USR-02 | `is_followed_by_me` is only sent with a token, from `user_follow`. | D+I |
| USR-03 | `GET /users/{username}/playlists` lists only that user's **public** playlists. | D+I |
| USR-04 | `GET /users/{username}/followers` and `/following` return user cards from `user_follow` joined to `app_user`, newest first (`followed_at DESC`). | D+I |
| USR-05 | User reads work without a token. | D+I |

## 6. Playlists

| ID | Requirement | Source |
|---|---|---|
| PL-01 | `POST /playlists` creates a playlist with `name`, optional `description`, `visibility` (default `public`), optional `cover_url` and optional `track_ids` (in order). | D+I |
| PL-02 | The playlist and its first tracks are inserted in one transaction. | D+I |
| PL-03 | Create answers `201 Created` with `Location`, `ETag` (the version), and the playlist, including `share_token` for the owner. | D+I |
| PL-04 | An unknown visibility or track answers `422` (`unknown_visibility`, `unknown_track`). | D+I |
| PL-05 | Limits: `name` 1 to 120 characters, `description` at most 2000, `cover_url` an http(s) URL of at most 500, at most 500 tracks per request. | I |
| PL-06 | `GET /playlists/{id}` returns the playlist from `playlist_summary` after checking `playlist_can_view`, with an `ETag`. | D+I |
| PL-07 | The playlist response has `playlist_id`, `name`, `description`, `visibility`, `cover_url`, `owner`, `is_collaborative`, `collaborator_count`, `track_count`, `duration_ms`, `follower_count`, `version`, `created_at`, `updated_at`. | D+I |
| PL-08 | `my_access` (`can_view`, `can_edit`, `is_following`) is only sent with a token. | D+I |
| PL-09 | An unlisted playlist opened by ID (not by share link) answers `404` to strangers. | D+I |
| PL-10 | `PATCH /playlists/{id}` changes `name`, `description`, `visibility` or `cover_url`. Only the owner can do it, and it answers `200 OK` with the playlist. | D+I |
| PL-11 | The design has `PATCH /playlists/{id}` send `If-Match`. The implementation checks `If-Match` on `PATCH` and `DELETE` only when it is sent. | D / I |
| PL-12 | `PATCH` on a playlist: `null` clears `description` and `cover_url`. `name` and `visibility` can't be blank. | I |
| PL-13 | Changing the visibility to `unlisted` or `private` removes the playlist from search and from the owner's profile at once. | D+I |
| PL-14 | `DELETE /playlists/{id}` answers `204`. Only the owner can do it, and the playlist's entries, collaborators and follows are deleted with it. | D+I |
| PL-15 | `GET /playlists/shared/{share_token}` opens a playlist by its share link and returns the same body as `GET /playlists/{id}`, without `share_token`. | D+I |
| PL-16 | An unknown or malformed share token answers `404`. | D+I |
| PL-17 | A share link only works while the playlist is `unlisted` or `public`. A `private` playlist is refused. | D+I |
| PL-18 | `GET /playlists/shared/{share_token}/tracks` lets someone with a share link read the entries. | I |
| PL-19 | `POST /playlists/{id}/share-token` (owner only) creates a new share link with `gen_random_uuid()`, answers `200 { "share_token": "…" }`, and the old link stops working at once. | D+I |
| PL-20 | `POST /playlists/{id}/copy` copies a playlist the caller can view into a new **private** playlist the caller owns, keeping the track order. The optional body `{ "name": … }` defaults to the original's name. It answers `201 Created`. | D+I |

### Versions and concurrency

| ID | Requirement | Source |
|---|---|---|
| PL-21 | `playlist.version` changes on every change to the playlist's track list, and is sent as `ETag: "<version>"`. | D+I |
| PL-22 | A write sends `If-Match: "<version>"`. If the version is current, the write goes ahead and the response carries the new `ETag`. | D+I |
| PL-23 | A stale `If-Match` answers `412 Precondition Failed`, and the client reloads. | D+I |
| PL-24 | A track-list write without `If-Match` answers `428 Precondition Required`. | I |
| PL-25 | `If-Match` uses strong comparison (RFC 9110). A weak tag (`W/"1"`) never matches, `*` matches any version, and the header can list several tags. | I |
| PL-26 | Clients compare versions and don't compute them. One request can raise the version by more than one, and the API reads the version back after a write. | D+I |
| PL-27 | A versioned update that changes 0 rows answers `404` if the row doesn't exist or isn't the caller's, and `412` otherwise. | D+I |

## 7. Playlist tracks

| ID | Requirement | Source |
|---|---|---|
| PLT-01 | A playlist holds **tracks** (specific recordings), not songs. The same track can appear more than once, so an entry is identified by its **position**. | D+I |
| PLT-02 | Every write needs `If-Match` and the owner or editor role. | D+I |
| PLT-03 | `GET /playlists/{id}/tracks` returns `playlist_id`, `version`, and the entries ordered by position. Each entry has `position`, `added_at`, `added_by`, and a `track` with `track_id`, `name`, `duration_ms`, `duration`, `artists` and `album`. | D+I |
| PLT-04 | `is_liked` on an entry is only sent with a token, from `user_saved_track`. | D+I |
| PLT-05 | `POST /playlists/{id}/tracks` appends by default. With `position` (1-based), it inserts there and pushes the following entries down. It answers `201` with `added`, `version` and a new `ETag`. | D+I |
| PLT-06 | When appending, the position is filled in by a database trigger. | D+I |
| PLT-07 | Adding: `403` when the caller isn't the owner or an editor, `412` for a stale version, `422` for an unknown track. | D+I |
| PLT-08 | Add limits: `track_ids` 1 to 500 entries, all positive. | I |
| PLT-09 | `POST /playlists/{id}/tracks/move` with `{ "from", "to" }` moves one entry, shifts the entries between, and answers `200` with the new version. It runs as a single statement. | D+I |
| PLT-10 | `DELETE /playlists/{id}/tracks/{position}` removes one entry, moves the following entries up one place, and answers `200 { "removed": { "position", "track_id" }, "version" }`. | D+I |
| PLT-11 | `DELETE /playlists/{id}/tracks?track_id=` removes every occurrence of a track, renumbers all positions, and answers `200 { "removed": <count>, "version" }`. | D+I |
| PLT-12 | Positions stay contiguous after every add, move and remove. | I |
| PLT-13 | Deletes and moves check `playlist_can_edit` first, because the database role trigger only guards inserts and changes of `added_by`. | D+I |
| PLT-14 | Invalid track-list changes are rejected (for example positions that don't exist). | I |

## 8. Collaborators

| ID | Requirement | Source |
|---|---|---|
| COL-01 | `GET /playlists/{id}/collaborators` lists `username`, `display_name`, `role` and `added_at`. Only the owner and the collaborators can see it. | D+I |
| COL-02 | `PUT /playlists/{id}/collaborators/{username}` with `{ "role" }` adds a collaborator or changes their role (an upsert). It answers `201` when the collaborator was added and `200` when the role changed. | D+I |
| COL-03 | Only the owner can add collaborators or change roles (`403` otherwise). | D+I |
| COL-04 | The role must be `editor` or `viewer` (`422` otherwise). | D+I |
| COL-05 | The owner can't also be a collaborator (`422`, for example "user 2 owns playlist 20 and can't also be a collaborator"). | D+I |
| COL-06 | `DELETE /playlists/{id}/collaborators/{username}` answers `204`. The owner can remove anyone, and a collaborator can remove themselves ("leave playlist"). | D+I |
| COL-07 | Entries a removed collaborator added stay, still showing them as `added_by`. | D+I |
| COL-08 | Adding a friend as a `viewer` is how a private playlist is shared with them. | D |

## 9. Library

| ID | Requirement | Source |
|---|---|---|
| LIB-01 | `PUT /me/tracks/{track_id}` likes a track and `DELETE` unlikes it. Both are idempotent and answer `204`. | D+I |
| LIB-02 | Liking an unknown track answers `404`. | D+I |
| LIB-03 | `GET /me/tracks` lists liked tracks, newest first, each with `saved_at` and a `track`. | D+I |
| LIB-04 | `GET /me/tracks/contains?ids=` answers, for each ID, whether the caller liked it (`{ "3674": true, … }`). | D+I |
| LIB-05 | `ids` takes 1 to 100 positive IDs. | I |
| LIB-06 | `GET /me/albums`, `PUT /me/albums/{album_id}` and `DELETE /me/albums/{album_id}` work the same way on `user_saved_album`. Each listed item is an `album_summary` card. | D+I |

## 10. Follows

| ID | Requirement | Source |
|---|---|---|
| FOL-01 | `PUT` / `DELETE /me/following/playlists/{id}` follows or unfollows a playlist (`playlist_follow`). | D+I |
| FOL-02 | `PUT` / `DELETE /me/following/artists/{slug}` follows or unfollows an artist (`artist_follow`). | D+I |
| FOL-03 | `PUT` / `DELETE /me/following/users/{username}` follows or unfollows a user (`user_follow`). | D+I |
| FOL-04 | Follow and unfollow are idempotent and answer `204`. | D+I |
| FOL-05 | Follower counts (`playlist.follower_count`, `artist.follower_count`, `app_user.follower_count` and `following_count`) are updated by triggers. The API never writes them. | D+I |
| FOL-06 | Following yourself answers `422` (`cannot_follow_self`). | D+I |
| FOL-07 | Following a playlist the caller can't view answers `404`. | D+I |
| FOL-08 | `GET /me/following/artists` and `GET /me/following/playlists` list what the caller follows. | D+I |
| FOL-09 | `GET /playlists/{id}/followers` lists who follows a playlist, with the same visibility rules as the playlist. | D+I |

## 11. Catalog

The catalog endpoints are read-only and public. Most are a single query against a view.

| ID | Requirement | Source |
|---|---|---|
| CAT-01 | `GET /artists` lists artists from `artist_summary`, filtered by `?type=`, `?country=` (two letters) and `?genre=`, and sorted by `?sort=name\|followers` (default `name`). | D+I |
| CAT-02 | `GET /artists/{slug}` returns the artist page: `artist_id`, `name`, `slug`, `artist_type`, `country_code`, `city`, `formed_year`, `disbanded_year`, `biography`, `image_url`, `follower_count`, `album_count`, `track_count`, `is_followed_by_me`. | D+I |
| CAT-03 | `GET /artists/{slug}/albums` lists every album the artist is on, collaborations included (`album_artist`), with the artist's `role`. It accepts `?type=studio\|live\|compilation\|soundtrack` and is ordered by `release_date NULLS LAST`. | D+I |
| CAT-04 | `GET /artists/{slug}/members` lists the line-up from `artist_member` joined to `person`, one row per stint (`person_id`, `name`, `roles`, `joined_year`, `left_year`). | D+I |
| CAT-05 | For a solo artist (`artist.person_id` set), `/members` answers with that one person. | D+I |
| CAT-06 | `GET /artists/{slug}/albums/{album_slug}` and `GET /albums/{id}` return the album from `album_summary`, plus `album_artist` for `artists`: `album_id`, `title`, `edition`, `slug`, `release_date`, `release_year`, `album_type`, `label`, `artist`, `artists`, `primary_genre`, `genres`, `track_count`, `disc_count`, `duration_ms`, `is_saved_by_me`. | D+I |
| CAT-07 | For a collaboration, `artist` is the credited name and `artists` lists every artist on the album. | D+I |
| CAT-08 | `GET /albums/{id}/tracks` lists the album's tracks from `track_detail`, ordered by `disc_number, track_number` (`disc_number`, `track_number`, `track_id`, `name`, `duration`, `composer`, `song_id`). | D+I |
| CAT-09 | `GET /tracks/{id}` returns one `track_detail` row: track, song, album, credited artist, `track_artists`, effective genre, composer and ISRC. | D+I |
| CAT-10 | `GET /songs/{id}` returns the song (the composition) with its writers from `credit`. | D+I |
| CAT-11 | `GET /songs/{id}/recordings` returns the song and every track that records it (`track_id`, `album_title`, `release_year`, `duration`). | D+I |
| CAT-12 | A credit shows `credited_as` when it is set (the name as printed). | D+I |
| CAT-13 | `GET /people/{id}` returns the person, the bands they were in (`artist_member`), the solo artist they are (`artist.person_id`), and their songwriting credits (`credit` → `song`). | D+I |
| CAT-14 | `GET /genres` returns the genre tree, nested through `parent_genre_id`. | D+I |
| CAT-15 | `GET /genres/{id}/albums` lists the albums in the genre and its subgenres (a recursive CTE). `?primary_only=true` matches only albums whose primary genre is in the tree. | D+I |
| CAT-16 | `GET /labels/{slug}` returns the label with its albums (`album_summary WHERE label_id = :id`). | D+I |
| CAT-17 | `is_followed_by_me` and `is_saved_by_me` are only sent with a token. | D+I |

## 12. Search

| ID | Requirement | Source |
|---|---|---|
| SRC-01 | `GET /search?q=` searches across artists, albums, songs, tracks, people, **public** playlists and users, through `search_catalog(q, limit)`. | D+I |
| SRC-02 | Matching is accent-insensitive (`?q=jose` finds "José José"), by substring and full text. | D+I |
| SRC-03 | `?types=` filters by kind: `artist`, `album`, `song`, `track`, `person`, `playlist`, `user`. An unknown kind answers `422`. | D+I |
| SRC-04 | Each result has `kind`, `id`, `name`, `context` and `score`, plus `slug` where the kind has one (artists, albums, and the username for users). The response also echoes `query`. | D+I |
| SRC-05 | `q` is required, not blank, and at most 100 characters. `limit` is 1 to 100. Bad input answers `422`. | I |

## 13. Reference data

| ID | Requirement | Source |
|---|---|---|
| REF-01 | `GET /reference/{name}` returns the allowed values from the lookup tables, as `[{ "code", "description" }]`. | D+I |
| REF-02 | `name` is one of `countries`, `artist-types`, `album-types`, `artist-roles`, `credit-roles`, `playlist-visibilities` or `playlist-roles`. | D+I |
| REF-03 | Reference data is served with a long `Cache-Control`. | D+I |

## 14. Catalog administration

| ID | Requirement | Source |
|---|---|---|
| ADM-01 | Catalog writes need a role: `curator` or higher in the implementation. | D+I |
| ADM-02 | An alternative to these endpoints is loading catalog data with SQL, as the seed does. | D |
| ADM-03 | `POST /admin/artists` creates an artist. The slug is generated from the name when it is omitted. | D+I |
| ADM-04 | `PATCH /admin/artists/{id}` edits an artist. The slug doesn't follow name changes: send `slug` to change it, or `null` to regenerate it from the name. | D+I |
| ADM-05 | `POST /admin/albums` creates an album with its artists (and genres) in **one transaction**, because the database checks at commit that an album lists its artists and has exactly one primary genre. | D+I |
| ADM-06 | Missing artists or a missing primary genre fail at commit with `integrity_constraint_violation`, answered as `422`. | D+I |
| ADM-07 | `POST /admin/albums/{id}/tracks` adds tracks with disc and track number, and an optional `genre_id`, which is added to the album's genres automatically. | D+I |
| ADM-08 | `POST /admin/songs/{id}/credits` adds credits, and the `track.composer` text of every recording of the song regenerates automatically. | D+I |
| ADM-09 | `PUT /admin/albums/{id}/genres` sets the album's complete set of genres, with exactly one `is_primary`. | D+I |
| ADM-10 | Artist fields: `name` (required, at most 120), `slug` (`^[a-z0-9]+(-[a-z0-9]+)*$`, at most 140), `artist_type`, `person_id`, `country_code`, `city` (at most 60), `formed_year` and `disbanded_year` (1000 to 2100), `biography` (at most 10000), `image_url` (http(s), at most 500), `musicbrainz_id` (UUID). | I |
| ADM-11 | Album fields: `title` (required, at most 160), `edition` (at most 80), `artist_id` (required), `slug`, `album_type`, `release_date`, `release_year` (must match `release_date`), `label_id`, `upc` (12 to 14 digits), `cover_url`, `musicbrainz_id`, `artists` (1 to 50, each with an optional `role`, default `main`), `genres` (at most 20). | I |
| ADM-12 | Track fields: `name` (required, at most 200), `disc_number` (default 1), `track_number`, `milliseconds` (required, positive), `song_id`, `genre_id`, `composer` (at most 1000), `bytes`, `isrc` (`^[A-Z]{2}[A-Z0-9]{3}[0-9]{7}$`). At most 200 tracks per request. | I |
| ADM-13 | Credit fields: `person_id` and `role` (required), `position` (default: after the existing credits), `credited_as` (at most 120). At most 50 credits per request. | I |
| ADM-14 | Rules enforced by the database: unique slugs, UPCs, ISRCs and MusicBrainz IDs; one artist per person; only solo artists link to a person; `disbanded_year` not before `formed_year`; unique disc and track number per album; positive durations; no duplicate credits; no overlapping band-membership stints; a genre that tracks use can't be removed from the album. | D+I |

## 15. Account administration

| ID | Requirement | Source |
|---|---|---|
| ADMU-01 | `PUT /admin/users/{username}/role` with `{ "role": "listener\|curator\|admin" }` sets an account's role and answers `{ user_id, username, role }`. Admins only. | I |
| ADMU-02 | Admins can't change their own role (`422 cannot_change_own_role`). | I |
| ADMU-03 | `PUT /admin/users/{username}/password` sets an account's password (8 to 72 characters), signs it out everywhere, and answers `204`. Admins only. | I |
| ADMU-04 | Setting a password lets an account without one (such as a seed user) log in, so the design's examples can be tried as that user. | I |
| ADMU-05 | An unknown username answers `404`. | I |

## 16. Errors

| ID | Requirement | Source |
|---|---|---|
| ERR-01 | Every error uses one envelope: `{ "error": { "code": "…", "message": "…" } }`. | D+I |
| ERR-02 | Validation errors also carry `details`: a list of `{ "field", "message" }`, with fields named as in the JSON. | I |
| ERR-03 | The API translates database errors by SQLSTATE instead of repeating the database's checks. | D+I |
| ERR-04 | `23505` unique_violation → `409 Conflict`. | D+I |
| ERR-05 | `23503` foreign_key_violation → `422` for a value in the body, `404` for a value in the URL. | D+I |
| ERR-06 | `23514` check_violation and `23P01` exclusion_violation → `422`. | D+I |
| ERR-07 | `23000` integrity_constraint_violation (raised by triggers) → `422`, passing the trigger's message through. | D+I |
| ERR-08 | `42501` insufficient_privilege → `403 Forbidden`, passing the trigger's message through. | D+I |
| ERR-09 | A versioned write that updates 0 rows → `412`, or `404` if the row doesn't exist. | D+I |
| ERR-10 | The constraint name in the error decides which field the response blames (for example `app_user_username_key` → `username`). | D+I |
| ERR-11 | Each known constraint maps to a specific code, field and message (`username_taken`, `email_taken`, `invalid_username`, `unknown_country`, `cannot_follow_self`, `unknown_visibility`, `unknown_track`, `unknown_album`, `slug_taken`, `album_exists`, `upc_taken`, `isrc_taken`, `multiple_primary_genres`, `genre_in_use`, `track_number_taken`, `duplicate_credit`, and others). New constraints are added to that map. | I |
| ERR-12 | An unknown constraint falls back on its SQLSTATE: `conflict`, `invalid_reference`, `constraint_violation`, `missing_value` (`23502`), `value_too_long` (`22001`), `value_out_of_range` (`22003`), `invalid_date` (`22007`/`22008`), `invalid_value` (`22P02`). | I |
| ERR-13 | Any other database error is a server error, not the client's fault. | I |
| ERR-14 | Generic codes by status: `bad_request` 400, `unauthorized` / `invalid_token` 401, `forbidden` 403, `not_found` 404, `method_not_allowed` 405, `not_acceptable` 406, `conflict` 409, `precondition_failed` 412, `payload_too_large` 413, `unsupported_media_type` 415, `validation_failed` 422, `precondition_required` 428, `too_many_requests` 429, `internal_error` 500, `service_unavailable` 503. | I |
| ERR-15 | Unknown endpoints, unsupported versions, wrong methods, wrong media types, and malformed or invalid bodies all answer with the envelope. Security errors (`401`, `403`) use it too. | I |
| ERR-16 | Spring's Problem Details and the whitelabel error page are off, so the envelope is the only error format. | I |

## 17. Database integration

| ID | Requirement | Source |
|---|---|---|
| DB-01 | The database model drives the design. Every endpoint names the table, view or function it reads or writes. | D+I |
| DB-02 | The database is the PostgreSQL of [db-postgresql-music](https://github.com/oggmont/db-postgresql-music), running in Docker as `music-db-postgres`. | I |
| DB-03 | The database enforces the rules (uniqueness, foreign keys, checks, role checks, deferred integrity rules). The API doesn't re-implement them. | D+I |
| DB-04 | Reads use the views `track_detail`, `album_summary`, `artist_summary` and `playlist_summary`, and the functions `playlist_can_view`, `playlist_can_edit` and `search_catalog`. | D+I |
| DB-05 | Triggers maintain derived data: follower and following counts, `updated_at`, playlist positions on append, playlist versions, slugs, and `track.composer`. The API never writes or recomputes them. | D+I |
| DB-06 | The API never changes the `public` schema. Schema and seed changes belong in db-postgresql-music. | I |
| DB-07 | The API owns an `auth` schema: `auth.role`, `auth.user_account` (role, password hash, `password_changed_at`) and `auth.refresh_token` (family, hash, issue, expiry, used and revoked times). | I |
| DB-08 | `auth` rows reference `app_user` with `ON DELETE CASCADE`. | I |
| DB-09 | Flyway manages only the `auth` schema (`baseline-on-migrate`, baseline version 1). On a database loaded by db-postgresql-music's init scripts, `V1` is recorded as the baseline instead of run. | I |
| DB-10 | The `auth` DDL exists in two places with identical content: `V1__auth_schema.sql` here and the end of `db/init/01_schema.sql` in db-postgresql-music. A change is a new `V2__…` migration here plus the same change there. Applied migrations are never edited. | I |
| DB-11 | `spring.sql.init.mode=never`: the schema never comes from `schema.sql`. | I |
| DB-12 | Data access is `JdbcClient` with explicit SQL and row mappers. No JPA. | I |
| DB-13 | List queries pick the page from base tables first, then join the heavy views. | I |
| DB-14 | Nullable parameters are bound with an explicit SQL type, and arrays are passed as `int[]` / `String[]` with `CAST(:x AS integer[])`. | I |
| DB-15 | The connection pool is Hikari (`music-db`), with 10 connections by default (`DB_POOL_SIZE`), a 5000 ms connection timeout, and `ApplicationName=api-spring-music`. | I |
| DB-16 | The live database is backed up before risky changes (`pg_dump -Fc`). Changes are tested on a throwaway database first. | I |
| DB-17 | Examples in the design use the real seed data (for example *Road Trip*, playlist 20, and *Abbey Road*, album 359), so they can be checked with `make psql`. | D |

## 18. Security

| ID | Requirement | Source |
|---|---|---|
| SEC-01 | Public without a token: `GET` on users, playlists, artists, albums, tracks, songs, people, genres, labels, search and reference; `POST /users`, `/auth/login`, `/auth/refresh` and `/auth/logout`; `/actuator/health` and `/actuator/info`; Swagger UI and `/v3/api-docs`. Everything else needs a token. Services still apply playlist visibility on public reads. | I |
| SEC-02 | CORS applies to `/api/**`: allowed origins from `APP_CORS_ALLOWED_ORIGINS` (default `http://localhost:3000,http://localhost:5173`); methods `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`; request headers `Authorization`, `Content-Type`, `If-Match`, `If-None-Match`; exposed headers `ETag`, `Location`, `WWW-Authenticate`; max age 1 hour. | I |
| SEC-03 | `APP_JWT_SECRET` is a Base64 HMAC key of at least 256 bits. It is required in Docker. Left empty in a local run, a random key is generated at each start. Rotating it signs everyone out. | I |
| SEC-04 | Secrets live only in `.env`, which is git-ignored. No password or password hash is ever committed. `Prompts.md` and `CLAUDE.md` are git-ignored. | I |
| SEC-05 | Before production: a long, unique super-user password; HTTPS behind a reverse proxy; rate limiting on `/auth/*`; a least-privilege database role (DML on `public`, owner of `auth`); CORS set to the real front end. | I |

## 19. Configuration

Configuration comes from environment variables (12-factor), normally in `.env`. Both `docker compose` and `./mvnw spring-boot:run` read it.

| ID | Variable | Default | Source |
|---|---|---|---|
| CFG-01 | `DB_HOST`, `DB_PORT` | `localhost`, `5432` (Docker: `DB_DOCKER_HOST` = `music-db-postgres`, port 5432) | I |
| CFG-02 | `DB_NAME`, `DB_USER`, `DB_PASSWORD` | `music_db`, `music`, `music` | I |
| CFG-03 | `DB_POOL_SIZE` | `10` | I |
| CFG-04 | `MUSIC_DB_NETWORK` | `music-db_default` | I |
| CFG-05 | `SERVER_PORT` | `8081` on the host (`8080` inside the container) | I |
| CFG-06 | `APP_JWT_SECRET` | none (see SEC-03) | I |
| CFG-07 | `APP_JWT_ISSUER`, `APP_JWT_AUDIENCE` | `api-spring-music` | I |
| CFG-08 | `APP_JWT_ACCESS_TOKEN_TTL`, `APP_JWT_REFRESH_TOKEN_TTL` | `15m`, `14d` | I |
| CFG-09 | `APP_ADMIN_USERNAME`, `APP_ADMIN_PASSWORD` | none (no super user) | I |
| CFG-10 | `APP_ADMIN_EMAIL`, `APP_ADMIN_DISPLAY_NAME` | `<username>@localhost`, the username | I |
| CFG-11 | `APP_CORS_ALLOWED_ORIGINS` | `http://localhost:3000,http://localhost:5173` | I |
| CFG-12 | `APP_API_DOCS_ENABLED` | `true` | I |
| CFG-13 | `JAVA_OPTS` | empty | I |
| CFG-14 | `MUSIC_DB_INIT_DIR` (tests) | `../db-postgresql-music/db/init` | I |
| CFG-15 | `.env.example` is the committed template, and `make env` creates `.env` from it with a random `APP_JWT_SECRET`. | – | I |

## 20. Deployment and operations

| ID | Requirement | Source |
|---|---|---|
| OPS-01 | The API runs in Docker as the `music-api` container (compose project `music-api`) and joins the database's network, reaching PostgreSQL as `music-db-postgres:5432` without going through the host. | I |
| OPS-02 | Host ports: the API uses **8081**, because Adminer uses 8080. The database is published on host port 55432 on this machine. | I |
| OPS-03 | The Dockerfile is multi-stage: a JDK stage builds the jar with the Maven wrapper, and a JRE-only runtime stage (`bellsoft/liberica-openjre-debian:25-cds`) runs it. | I |
| OPS-04 | The jar is extracted into layers (dependencies, loader, snapshot dependencies, application), so a code change only rebuilds the application layer. | I |
| OPS-05 | A Java 25 AOT cache is recorded during the build (a training run with `-Dspring.context.exit=onRefresh`) to shorten startup (about 5 s). Nothing may connect to the database during that run: Flyway is disabled there. | I |
| OPS-06 | The container runs as a non-root user (uid 10001). | I |
| OPS-07 | The container's health check is `/actuator/health/readiness`, every 15 s (start period 40 s). | I |
| OPS-08 | The heap is sized from the container's memory (`-XX:MaxRAMPercentage=75`), with a 768 MB limit in compose. | I |
| OPS-09 | The container restarts `unless-stopped`. | I |
| OPS-10 | Spring MVC runs on virtual threads. | I |
| OPS-11 | Actuator exposes `health`, `info` and `metrics`. Liveness and readiness probes are on. Health details are shown to admins only, and `/actuator/info` includes Java, OS and build information. | I |
| OPS-12 | Make targets: `help`, `env`, `up`, `down`, `logs`, `restart`, `ps`, `build`, `test`, `verify`, `run` and `run-testdb` (a local run on a throwaway Testcontainers database). | I |
| OPS-13 | After `make reset` in db-postgresql-music, the API is restarted so it sets the super user's password again. | I |
| OPS-14 | The Docker build context contains only the Maven wrapper, `pom.xml` and `src/` (see `.dockerignore`). | I |

## 21. API documentation

| ID | Requirement | Source |
|---|---|---|
| DOC-01 | OpenAPI is served at `/v3/api-docs`, with Swagger UI at `/swagger-ui.html` (tags sorted alphabetically, operations by method). Both can be turned off with `APP_API_DOCS_ENABLED=false`. | I |
| DOC-02 | OpenAPI declares a bearer (JWT) security scheme, and Swagger UI's **Authorize** button accepts an access token. | I |
| DOC-03 | Documented schemas are `snake_case`, and a `PartialUpdate<T>` body is documented as `T`. Request and response records have unique simple names, so their schemas don't overwrite each other. | I |
| DOC-04 | Documented paths use the concrete version (`/api/v1/...`), so "Try it out" works. | I |
| DOC-05 | API-Design.md is the source of truth: a design change is made there first, and the README is updated only when the overview changes. | D |
| DOC-06 | After an endpoint change, the implementation's README endpoint table and the OpenAPI annotations are kept in sync. | I |

## 22. Testing

| ID | Requirement | Source |
|---|---|---|
| TST-01 | Unit tests are `*Test` (Surefire, `./mvnw test`), and integration tests are `*IT` (Failsafe, `./mvnw verify`). | I |
| TST-02 | Integration tests run against a throwaway PostgreSQL 17 (Testcontainers) loaded with the real schema and seed from `../db-postgresql-music/db/init`, through MockMvc with security on. They need Docker. | I |
| TST-03 | Tests create their own users and playlists. They don't assert absolute counts that another test could change, and never modify seed data that other tests read (Abbey Road 359, Road Trip 20). | I |
| TST-04 | There are 68 tests (14 unit, 54 integration), and they all pass. | I |
| TST-05 | The tests cover the scenarios below. | I |

**Covered scenarios**

- **Playlists:** create answers with `Location`, `ETag` and the share token for the owner; visibility decides who can open a playlist; track-list writes need the current version; add at a position, move and remove keep positions contiguous; invalid track-list changes are rejected; keyset paging of entries; `PATCH` changes only the fields sent and checks `If-Match` when given; share links open unlisted playlists until regenerated; copying creates a private playlist owned by the caller; owners and editors change tracks, viewers only look; the *Road Trip* design examples; the feed shows entries of followed and shared playlists; only the owner deletes a playlist.
- **Auth:** reasonable passwords are accepted, and short, blank and overlong ones rejected; the super user is created at startup as an admin; login returns an OAuth-style token response; a wrong password and an unknown user get the same answer; seed users can't log in until they have a password; requests without a token, or with a bad one, are rejected; sign-up creates a profile that can log in; sign-up answers with `Location` and rejects taken names and bad input; refresh tokens rotate and reuse revokes the session; logout ends the session; changing the password needs the current one and ends other sessions; deleting the account invalidates its tokens.
- **Catalog:** artist page; artists filtered and sorted; discography with the artist's role, in release order; band line-up with one row per stint, and solo artists answered with their person; album by artist and album slug; album track listing in disc and track order; track, song and recordings; credits show the name as printed; person page with bands and credits; genre tree and albums including subgenres; label with its albums.
- **Search and reference:** search filters by kind; search is accent-insensitive; search rejects bad input; reference data is cacheable.
- **Library and follows:** liking is idempotent and `contains` answers for each track; liked tracks are newest first with a keyset cursor; saved albums; following an artist updates its follower count; following users; following playlists respects visibility.
- **Admin:** roles guard the admin endpoints; admins manage roles and passwords but not their own role; artists get generated slugs that don't follow renames; albums are created with their artists and genres in one transaction; credits regenerate the composer text of every recording.
- **Errors and helpers:** unknown endpoints and unsupported versions; wrong method and media type; malformed and invalid bodies; durations format like the `track_detail` view and show hours for long tracks; known constraints name the code and field; unknown constraints fall back on the SQLSTATE; trigger messages are passed through; other errors are not the client's fault; `If-Match` parsing (absent header, strong tags, lists, wildcards, weak tags, the quoted-version ETag); field paths use the JSON names.

## 23. Technology and code constraints

| ID | Requirement | Source |
|---|---|---|
| TECH-01 | Spring Boot 4.1 (4.1.2), Spring Framework 7, Spring Security 7, Java 25, Maven through the wrapper (`./mvnw`). | I |
| TECH-02 | Dependencies: Spring Web MVC, Validation, Security, OAuth2 Resource Server (Nimbus JWT), JDBC, Flyway (with `flyway-database-postgresql`), the PostgreSQL driver, Actuator, springdoc-openapi 3.2.0; for tests, Testcontainers (JUnit Jupiter and PostgreSQL) and the Spring Boot test starters. | I |
| TECH-03 | Jackson 3 (`tools.jackson.*`) for the API. springdoc still uses Jackson 2, so it gets its own snake_case model resolver. | I |
| TECH-04 | JDK 25 is required for local builds. On this machine it is Temurin 25 in `~/.jdks/jdk-25.0.5+8`, with `JAVA_HOME` set before Maven commands; the system default stays Java 19. | I |
| TECH-05 | Packages by feature: `auth`, `account`, `user`, `playlist`, `library`, `follow`, `catalog`, `search`, `reference`, `admin`, plus `common` and `config`. | I |
| TECH-06 | Layers: controller (HTTP) → service (rules, transactions) → repository (SQL). Pure catalog reads go controller → repository. | I |
| TECH-07 | Errors are thrown as `ApiException` (factories for 400, 401, 403, 404, 409, 412, 422 and 428). Constraint violations propagate to `DatabaseErrors`. | I |
| TECH-08 | The caller is injected with `@CurrentUser Caller` (`@CurrentUser @Nullable Caller` on public endpoints). | I |
| TECH-09 | `PATCH` bodies are `@Valid @RequestBody PartialUpdate<Body>`, using `patch.has("json_name")` and `patch.rejectNulls(...)`. | I |
| TECH-10 | Null-safety: every package is `@NullMarked` (JSpecify), and nullable values are marked `@Nullable`. | I |
| TECH-11 | A `Clock` bean (UTC) provides the time, so token lifetimes can be tested with a fixed clock. | I |
| TECH-12 | The build adds build information for `/actuator/info` (with no build time, since builds are reproducible) and runs the configuration processor. | I |
| TECH-13 | Changes aren't committed or pushed unless the user asks. | I |

## 24. Design vs. implementation

Where the implementation departs from, or adds to, API-Design.md:

| Topic | API-Design.md | api-spring-music |
|---|---|---|
| Token issuer | An external login provider. | The API issues its own JWTs and refresh tokens (`/auth/login`, `/auth/refresh`, `/auth/logout`). |
| Sign-up | `POST /users` with a token; `sub` becomes `auth_subject`. | `POST /users` without a token, with a `password`. |
| Passwords | The database never sees passwords. | bcrypt hashes in the API-owned `auth` schema; `public` still has none. |
| Roles | Not in the schema yet; something like `app_user.role` is suggested. | `listener` < `curator` < `admin` in `auth.user_account`, read on every request. |
| Catalog administration | Needs "an admin role". | Needs `curator` or higher. |
| Share-link entries | Only the playlist's details can be read by share link. | Adds `GET /playlists/shared/{share_token}/tracks`. |
| Missing `If-Match` | Writes "should send" it. | Track-list writes answer `428` without it. Playlist `PATCH` and `DELETE` check it only when it is sent. |
| Library cursor | `?before=<saved_at>`. | Adds `before_id` to break ties. |
| Error body | `code` and `message`. | Adds `details` (`field`, `message`) for validation errors. |
| New endpoints | – | `PUT /me/password`, `PUT /admin/users/{username}/role`, `PUT /admin/users/{username}/password`. |
| `GET /me` | Profile plus `email`. | Also `role`. |
| Database | Read only through its tables, views and functions. | Also an `auth` schema and the `oscargm` row, so db-postgresql-music's `make verify` reports 8 rows in `app_user` instead of 7. |

## 25. Open items

| ID | Item | Source |
|---|---|---|
| OPEN-01 | **Admin role.** The design marks the role model as missing from the schema. The implementation solved it in the `auth` schema; the design should be updated to match. | D / I |
| OPEN-02 | **Username renames.** Usernames are permanent because they appear in URLs. Allowing renames would also need redirects. | D+I |
| OPEN-03 | **Missing design files.** API-Design.md links to `data-model.md`, `music-db-schema.jpg` and `../db/queries/examples.sql`, which live in db-postgresql-music, not in this repository. | D |
| OPEN-04 | **Weak super-user password.** The current super-user password is short and logs a warning at startup. Change it with `PUT /me/password` before the API is exposed beyond this machine. | I |
| OPEN-05 | **Production hardening.** HTTPS, rate limiting on `/auth/*`, a least-privilege database role and the real CORS origin are not set up yet (SEC-05). | I |
