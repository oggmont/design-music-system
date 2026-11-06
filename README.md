# design-music-restapi

Design documentation for the **music-db REST API**: the HTTP interface of a playlist platform built on a PostgreSQL music catalog (artists, albums, songs, tracks, people, genres, labels) and the social layer around it (users, playlists, collaborators, libraries, follows).

This repository holds design only, with no implementation. The full specification is in **[API-Design.md](API-Design.md)**, which is the source of truth. This README is a summary and a way into it.

---

## Contents

- [What's in this repo](#whats-in-this-repo)
- [Design at a glance](#design-at-a-glance)
- [Resources](#resources)
- [Conventions](#conventions)
- [Access rules](#access-rules)
- [Errors](#errors)
- [Open items](#open-items)
- [Related material](#related-material)

---

## What's in this repo

| File | Purpose |
|---|---|
| [API-Design.md](API-Design.md) | The complete API specification: conventions, every endpoint with request and response examples, the SQL behind each one, and how database errors map to HTTP. |
| [README.md](README.md) | This overview. |

---

## Design at a glance

- **The database model drives the design.** Every endpoint lists the table, view or function it reads or writes (`playlist_summary`, `track_detail`, `album_summary`, `search_catalog(...)`, `playlist_can_view(...)`, …). Most read endpoints are a single query against a view.
- **The database enforces the rules.** Uniqueness, foreign keys, check constraints, role checks and deferred integrity rules all live in PostgreSQL. The API translates their errors (by SQLSTATE and constraint name) into HTTP responses and doesn't re-implement the checks.
- **Triggers keep counters right.** `follower_count`, `following_count`, `updated_at`, playlist positions on append, and `track.composer` are maintained by triggers. The API never writes them.
- **Optimistic concurrency for playlists.** `playlist.version` is exposed as an `ETag`, and writes send `If-Match`. A stale version gets `412 Precondition Failed`.
- **Private things stay invisible.** A playlist the caller can't see returns `404`, not `403`, so its existence isn't leaked.
- **Examples use real seed data.** IDs, names and values in the spec (e.g. *Road Trip*, playlist 20; *Abbey Road*, album 359) come from the seed, so you can check them against a local database with `make psql`.

---

## Resources

All paths are relative to the base URL **`/api/v1`**. Each group links to its section in the spec.

| Group | Main paths | Summary |
|---|---|---|
| [Account](API-Design.md#account-me) | `POST /users`, `/me`, `/me/playlists`, `/me/feed` | Sign up after first login, edit or delete your profile, list your playlists, see a feed of recent additions. |
| [Users](API-Design.md#users) | `/users/{username}`, `…/playlists`, `…/followers`, `…/following` | Public profiles and their public playlists and social graph. |
| [Playlists](API-Design.md#playlists) | `/playlists`, `/playlists/{id}`, `/playlists/shared/{share_token}`, `…/share-token`, `…/copy` | Create, read, edit, delete, share by link, rotate the link, copy. |
| [Playlist tracks](API-Design.md#playlist-tracks) | `/playlists/{id}/tracks`, `…/tracks/move`, `…/tracks/{position}` | Ordered entries: add (append or insert), move, remove one entry or every occurrence of a track. |
| [Collaborators](API-Design.md#collaborators) | `/playlists/{id}/collaborators/{username}` | Add editors or viewers, change roles, remove or leave. |
| [Library](API-Design.md#library) | `/me/tracks`, `/me/tracks/contains`, `/me/albums` | Liked tracks and saved albums (idempotent `PUT` / `DELETE`). |
| [Follows](API-Design.md#follows) | `/me/following/{playlists,artists,users}/…`, `/playlists/{id}/followers` | Follow and unfollow playlists, artists and users. |
| [Catalog](API-Design.md#catalog) | `/artists`, `/albums`, `/tracks`, `/songs`, `/people`, `/genres`, `/labels` | Read-only, public browsing of the music catalog. |
| [Search](API-Design.md#search) | `/search?q=` | One accent-insensitive search across every kind of resource. |
| [Reference data](API-Design.md#reference-data) | `/reference/{name}` | Allowed values for dropdowns and validation (countries, roles, visibilities…). |
| [Catalog administration](API-Design.md#catalog-administration) | `/admin/…` | Proposed catalog writes. Blocked on an admin role (see [Open items](#open-items)). |

The full table of methods, paths and required auth is the [endpoint index](API-Design.md#endpoint-index).

### Domain vocabulary

- **Song**: a composition, with its writing credits.
- **Track**: one recording of a song on one album. Playlists and libraries hold **tracks**, not songs.
- **Playlist entry**: a track at a **position**. The same track can appear more than once, so entries are identified by position.
- **Artist**: a group or a solo act. A solo artist links to a **person**, and groups list their members (one row per stint).

---

## Conventions

Summarised from [Conventions](API-Design.md#conventions):

| Topic | Rule |
|---|---|
| Format | JSON, UTF-8, `snake_case` field names that match the column names. |
| Time | ISO 8601 in UTC, e.g. `"2026-11-05T20:43:11Z"`. |
| Durations | `duration_ms`, plus a display string (`"4:20"`) where a view provides one. |
| Auth | `Authorization: Bearer <token>` from an external identity provider. The token's `sub` maps to `app_user.auth_subject`. The database never stores passwords. |
| URL keys | `username` for users, `slug` for artists and labels, numeric IDs for playlists, albums, tracks, songs, people and genres. |
| Pagination | `?limit=` (default 20, max 100) and `?offset=`. Responses look like `{ items, limit, offset, total }`. Lists that change often also accept a keyset cursor. |
| Concurrency | `ETag` / `If-Match` on playlist writes, using `playlist.version`. Compare versions; don't compute them. |

Catalog reads and public-playlist reads work without a token. Everything under `/me` and every write needs one.

---

## Access rules

| `visibility` | Who can open it | Listed on profile and in search |
|---|---|---|
| `public` | everyone | yes |
| `unlisted` | anyone with the share link, plus the owner and collaborators | no |
| `private` | the owner and collaborators | no |

| Role | Can do |
|---|---|
| **owner** | Everything: edit details and visibility, manage collaborators, rotate the share link, delete. Only the owner receives `share_token`. |
| **editor** | Add, move and remove tracks. |
| **viewer** | Open the playlist, even when it's private. |

---

## Errors

Every error uses one envelope:

```json
{ "error": { "code": "forbidden", "message": "Only the owner and editors can add tracks to this playlist." } }
```

| Database condition | HTTP |
|---|---|
| `23505` unique_violation | `409 Conflict` |
| `23503` foreign_key_violation | `422` (bad body value) or `404` (bad URL value) |
| `23514` check_violation, `23P01` exclusion_violation, `23000` integrity_constraint_violation | `422` |
| `42501` insufficient_privilege | `403 Forbidden` |
| No rows updated by a versioned write | `412 Precondition Failed` (or `404` if the row is gone) |

The full table is in [Mapping database errors to HTTP](API-Design.md#mapping-database-errors-to-http).

---

## Open items

- **Admin role.** Catalog writes (`/admin/…`) need a permission model the schema doesn't have yet. `is_verified` marks editorial and artist accounts, not permissions. The spec suggests adding something like `app_user.role` (`listener`, `curator`, `admin`), or loading catalog data with SQL as the seed does.
- **Username renames.** Usernames are permanent because they appear in URLs. Allowing renames would also need redirects.

---

## Related material

The spec refers to files from the **music-db** database project. They are not in this repository:

- `data-model.md`: description of the data model
- `music-db-schema.jpg`: schema diagram
- `db/queries/examples.sql`: example queries mentioned by number (e.g. example 10, the recursive genre CTE; example 23, the feed)

Keep those links working by viewing this repository next to the database project, or by copying those files in here.

---

## Contributing

1. Change the design in [API-Design.md](API-Design.md) first. It's the source of truth.
2. Update this README only if the change affects the overview (a new resource group, a new convention, a new open item).
3. Keep examples based on the real seed data, so they can be checked with `make psql`.
