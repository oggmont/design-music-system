# API Design

A REST API for the music-db playlist platform, designed around the current database model ([data-model.md](data-model.md), [music-db-schema.jpg](music-db-schema.jpg)). Each endpoint lists what it reads or writes in the database. The examples use the real seed data, so you can try them with `make psql`.

**Contents**

- [Conventions](#conventions)
- [Endpoint index](#endpoint-index)
- [Account (`/me`)](#account-me)
- [Users](#users)
- [Playlists](#playlists)
- [Playlist tracks](#playlist-tracks)
- [Collaborators](#collaborators)
- [Library](#library)
- [Follows](#follows)
- [Catalog](#catalog)
- [Search](#search)
- [Reference data](#reference-data)
- [Catalog administration](#catalog-administration)
- [Mapping database errors to HTTP](#mapping-database-errors-to-http)

---

## Conventions

### Base URL and format

- Base URL: `/api/v1`. JSON in and out, UTF-8, `snake_case` fields (the same names as the columns).
- Timestamps are ISO 8601 in UTC: `"2026-11-05T20:43:11Z"`.
- Durations are returned as `duration_ms` (from `milliseconds`), and also as a display string where the views provide one (`"4:20"`).

### Authentication

- Requests carry `Authorization: Bearer <token>` from your login provider (Auth0, Cognito, Keycloak, Supabase…).
- The API reads the token's `sub` claim and looks up `app_user.auth_subject` to find the user. The database never sees passwords.
- Catalog and public-playlist reads work without a token. Everything under `/me`, and every write, needs one.
- In this document, **"the caller"** is the user the token belongs to.

### Identifiers in URLs

| Resource | URL key | Example |
|---|---|---|
| User | `username` | `/users/ana.garcia` |
| Artist | `slug` | `/artists/the-beatles` |
| Album | `album_id`, or artist slug + album slug | `/albums/359`, `/artists/the-beatles/albums/abbey-road` |
| Playlist | `playlist_id` | `/playlists/20` |
| Track, song, person, genre | numeric ID | `/tracks/3668`, `/songs/3281` |
| Label | `slug` | `/labels/apple` |

### Pagination

List endpoints accept `?limit=` (default 20, max 100) and `?offset=`, and return:

```json
{ "items": [ ... ], "limit": 20, "offset": 0, "total": 193 }
```

Long lists that change often (playlist entries, a user's library) also accept a keyset cursor (`?after_position=` or `?before=<saved_at>`), which stays stable while items are added.

### Playlist versions (concurrent edits)

`playlist.version` changes on every change to a playlist's track list. The API sends it as an `ETag`:

```
GET /api/v1/playlists/20
ETag: "1"
```

Every write to a playlist or its tracks should send `If-Match: "<version>"`:

- If the version is still current, the write goes ahead, and the response carries the new `ETag`.
- If not, the API answers `412 Precondition Failed`, and the client reloads.

Compare versions; don't compute them. One request can raise the version by more than one (for example, removing a track and closing the gap is two writes).

### Errors

```json
{
  "error": {
    "code": "forbidden",
    "message": "Only the owner and editors can add tracks to this playlist."
  }
}
```

The database enforces most rules itself. See [Mapping database errors to HTTP](#mapping-database-errors-to-http) for how its errors become status codes.

### Visibility and access

| `visibility` | Who can open it | Listed on profile and in search |
|---|---|---|
| `public` | everyone | yes |
| `unlisted` | anyone with the share link (`share_token`), plus the owner and collaborators | no |
| `private` | the owner and collaborators | no |

- The API uses `playlist_can_view(playlist_id, user_id)` and `playlist_can_edit(playlist_id, user_id)` for these checks.
- For a playlist the caller can't see, answer **404**, not 403, so private playlists don't reveal that they exist.
- `share_token` is a secret. Only the owner ever receives it.

---

## Endpoint index

| Method | Path | What it does | Auth |
|---|---|---|---|
| **Account** | | | |
| `POST` | `/users` | Sign up: create the caller's profile | token |
| `GET` | `/me` | The caller's profile | ✓ |
| `PATCH` | `/me` | Edit display name, bio, avatar, country | ✓ |
| `DELETE` | `/me` | Delete the account and everything it owns | ✓ |
| `GET` | `/me/playlists` | Playlists the caller owns or collaborates on | ✓ |
| `GET` | `/me/feed` | Recent additions to playlists the caller follows or edits | ✓ |
| **Users** | | | |
| `GET` | `/users/{username}` | Public profile | – |
| `GET` | `/users/{username}/playlists` | Their public playlists | – |
| `GET` | `/users/{username}/followers` | Who follows them | – |
| `GET` | `/users/{username}/following` | Users they follow | – |
| **Playlists** | | | |
| `POST` | `/playlists` | Create a playlist | ✓ |
| `GET` | `/playlists/{id}` | Playlist details | – / ✓ |
| `PATCH` | `/playlists/{id}` | Rename, describe, change visibility or cover | owner |
| `DELETE` | `/playlists/{id}` | Delete a playlist | owner |
| `GET` | `/playlists/shared/{share_token}` | Open a playlist by its share link | – |
| `POST` | `/playlists/{id}/share-token` | Make a new share link (the old one stops working) | owner |
| `POST` | `/playlists/{id}/copy` | Copy a playlist into the caller's account | ✓ |
| **Playlist tracks** | | | |
| `GET` | `/playlists/{id}/tracks` | Entries in order | – / ✓ |
| `POST` | `/playlists/{id}/tracks` | Add tracks (at the end or at a position) | owner, editor |
| `POST` | `/playlists/{id}/tracks/move` | Move an entry to another position | owner, editor |
| `DELETE` | `/playlists/{id}/tracks/{position}` | Remove one entry | owner, editor |
| `DELETE` | `/playlists/{id}/tracks?track_id=` | Remove every occurrence of a track | owner, editor |
| **Collaborators** | | | |
| `GET` | `/playlists/{id}/collaborators` | List collaborators | owner, collaborators |
| `PUT` | `/playlists/{id}/collaborators/{username}` | Add a collaborator or change their role | owner |
| `DELETE` | `/playlists/{id}/collaborators/{username}` | Remove a collaborator (or leave) | owner, self |
| **Library** | | | |
| `GET` | `/me/tracks` | Liked tracks | ✓ |
| `PUT` | `/me/tracks/{track_id}` | Like a track | ✓ |
| `DELETE` | `/me/tracks/{track_id}` | Unlike | ✓ |
| `GET` | `/me/tracks/contains?ids=` | Which of these tracks are liked | ✓ |
| `GET` | `/me/albums` | Saved albums | ✓ |
| `PUT` | `/me/albums/{album_id}` | Save an album | ✓ |
| `DELETE` | `/me/albums/{album_id}` | Remove a saved album | ✓ |
| **Follows** | | | |
| `PUT` / `DELETE` | `/me/following/playlists/{id}` | Follow / unfollow a playlist | ✓ |
| `PUT` / `DELETE` | `/me/following/artists/{slug}` | Follow / unfollow an artist | ✓ |
| `PUT` / `DELETE` | `/me/following/users/{username}` | Follow / unfollow a user | ✓ |
| `GET` | `/me/following/artists` | Artists the caller follows | ✓ |
| `GET` | `/me/following/playlists` | Playlists the caller follows | ✓ |
| `GET` | `/playlists/{id}/followers` | Who follows a playlist | – / ✓ |
| **Catalog** | | | |
| `GET` | `/artists` | List and filter artists | – |
| `GET` | `/artists/{slug}` | Artist page | – |
| `GET` | `/artists/{slug}/albums` | Discography, including collaborations | – |
| `GET` | `/artists/{slug}/members` | Band line-up | – |
| `GET` | `/artists/{slug}/albums/{album_slug}` | Album by URL | – |
| `GET` | `/albums/{id}` | Album details | – |
| `GET` | `/albums/{id}/tracks` | Track listing | – |
| `GET` | `/tracks/{id}` | Track details | – |
| `GET` | `/songs/{id}` | Song (composition) with credits | – |
| `GET` | `/songs/{id}/recordings` | Every track of the song | – |
| `GET` | `/people/{id}` | Person: bands and songwriting credits | – |
| `GET` | `/genres` | Genre tree | – |
| `GET` | `/genres/{id}/albums` | Albums in a genre and its subgenres | – |
| `GET` | `/labels/{slug}` | Label with its albums | – |
| **Search and reference** | | | |
| `GET` | `/search?q=` | One search box across everything | – |
| `GET` | `/reference/{name}` | Allowed values (countries, visibilities, roles…) | – |

"– / ✓" means it works without a token for public playlists, and needs one (and access) otherwise.

---

## Account (`/me`)

### `POST /users` — sign up

Called once after the user's first login with the provider. The token's `sub` becomes `auth_subject`.

```http
POST /api/v1/users
Authorization: Bearer eyJ...
Content-Type: application/json

{
  "username": "sofia.lopez",
  "display_name": "Sofía López",
  "email": "sofia.lopez@example.com",
  "country_code": "ES"
}
```

```http
201 Created
Location: /api/v1/users/sofia.lopez

{
  "user_id": 8,
  "username": "sofia.lopez",
  "display_name": "Sofía López",
  "country_code": "ES",
  "bio": null,
  "avatar_url": null,
  "is_verified": false,
  "follower_count": 0,
  "following_count": 0,
  "created_at": "2026-11-05T21:02:14Z"
}
```

- **Database:** `INSERT INTO app_user (username, display_name, email, auth_subject, country_code) ... RETURNING ...`
- **Errors:**
  - `409 username_taken` / `409 email_taken`: unique index `app_user_username_key` or `app_user_email_key`.
  - `422`: a username that doesn't match `^[a-z0-9_.]{3,30}$`, or an unknown country code.

### `GET /me`

Same shape as above, plus `email`, which is only ever shown to its owner.

### `PATCH /me`

Send only the fields that change: `display_name`, `bio`, `avatar_url`, `country_code`. Usernames are permanent, because they appear in URLs. If you allow renames, add redirects.

```http
PATCH /api/v1/me
{ "bio": "Road trips, boleros and French house." }
```

`200 OK` with the updated profile. `updated_at` is set by a trigger.

### `DELETE /me`

`204 No Content`. One `DELETE FROM app_user` takes the caller's owned playlists, collaborations, library and follows with it (`ON DELETE CASCADE`). Everyone's follower counts are corrected by triggers. Tracks the user added to other people's playlists stay, with `added_by = NULL`.

### `GET /me/playlists`

Playlists the caller owns or collaborates on, with the caller's role on each.

```json
{
  "items": [
    {
      "playlist_id": 20,
      "name": "Road Trip",
      "visibility": "public",
      "owner": { "username": "ana.garcia", "display_name": "Ana García" },
      "my_role": "owner",
      "is_collaborative": true,
      "track_count": 7,
      "duration_ms": 2129000,
      "follower_count": 3,
      "version": 1
    }
  ],
  "limit": 20, "offset": 0, "total": 1
}
```

**Database:** `playlist_summary WHERE owner_id = :me`, `UNION` the playlists where `playlist_collaborator.user_id = :me` (`my_role` comes from `playlist_collaborator.role`).

### `GET /me/feed`

The newest entries in playlists the caller follows or collaborates on. This is example 23 in [examples.sql](../db/queries/examples.sql), ordered by `playlist_track.added_at DESC`.

---

## Users

### `GET /users/{username}`

```http
GET /api/v1/users/ana.garcia
```

```json
{
  "user_id": 2,
  "username": "ana.garcia",
  "display_name": "Ana García",
  "bio": "Road trips and Sunday boleros.",
  "country_code": "MX",
  "avatar_url": null,
  "is_verified": false,
  "follower_count": 2,
  "following_count": 2,
  "is_followed_by_me": false
}
```

`is_followed_by_me` is only sent with a token. It comes from `EXISTS (SELECT 1 FROM user_follow WHERE follower_id = :me AND followed_id = 2)`.

### `GET /users/{username}/playlists`

Their **public** playlists only, from `playlist_summary WHERE owner_username = :username AND visibility = 'public'`. Their own call (`/me/playlists`) shows everything.

### `GET /users/{username}/followers` and `/following`

Lists of user cards from `user_follow` joined to `app_user`, newest first (`followed_at DESC`).

---

## Playlists

### `POST /playlists` — create

```http
POST /api/v1/playlists
Authorization: Bearer eyJ...   (ana.garcia)

{
  "name": "Sunday Morning",
  "description": "Coffee and vinyl.",
  "visibility": "private",
  "track_ids": [3674, 3669, 4037]
}
```

```http
201 Created
Location: /api/v1/playlists/25
ETag: "2"

{
  "playlist_id": 25,
  "name": "Sunday Morning",
  "description": "Coffee and vinyl.",
  "visibility": "private",
  "share_token": "3f0d6a8e-9b1c-4d8e-a6f2-7c51e0b9d4aa",
  "cover_url": null,
  "owner": { "user_id": 2, "username": "ana.garcia", "display_name": "Ana García" },
  "is_collaborative": false,
  "track_count": 3,
  "duration_ms": 685000,
  "follower_count": 0,
  "version": 2,
  "created_at": "2026-11-05T21:10:00Z",
  "updated_at": "2026-11-05T21:10:00Z"
}
```

- **Fields:** `track_ids` is optional. `visibility` defaults to `public`.
- **Database:** both statements run in one transaction.
  ```sql
  INSERT INTO playlist (name, description, owner_id, visibility)
  VALUES (:name, :description, :me, :visibility) RETURNING playlist_id;

  INSERT INTO playlist_track (playlist_id, track_id)        -- position and added_by are filled in
  SELECT :playlist_id, t.id FROM unnest(:track_ids::int[]) WITH ORDINALITY AS t(id, n) ORDER BY n;
  ```
- **Errors:** `422` for an unknown visibility (`playlist_visibility_fkey`) or a track that doesn't exist (`playlist_track_track_id_fkey`).

### `GET /playlists/{id}`

```http
GET /api/v1/playlists/20
```

```http
200 OK
ETag: "1"

{
  "playlist_id": 20,
  "name": "Road Trip",
  "description": "Songs for the drive. Add yours!",
  "visibility": "public",
  "cover_url": null,
  "owner": { "user_id": 2, "username": "ana.garcia", "display_name": "Ana García" },
  "is_collaborative": true,
  "collaborator_count": 2,
  "track_count": 7,
  "duration_ms": 2129000,
  "follower_count": 3,
  "version": 1,
  "my_access": { "can_view": true, "can_edit": false, "is_following": true },
  "created_at": "2026-11-05T20:43:11Z",
  "updated_at": "2026-11-05T20:43:11Z"
}
```

- **Database:** `playlist_summary WHERE playlist_id = 20`, after checking `playlist_can_view(20, :me)`.
- **Responses:**
  - `my_access` is only sent with a token (shown here for kenji.watanabe, who follows the playlist).
  - **404** if the caller can't view it. An unlisted playlist opened by ID (not by share link) also returns 404 to strangers.
  - `share_token` is included only when the caller is the owner.

### `PATCH /playlists/{id}` — edit

Change any of `name`, `description`, `visibility` or `cover_url`. Only the owner can do this. Editors change tracks, not the playlist's details.

```http
PATCH /api/v1/playlists/20
If-Match: "1"

{ "name": "Road Trip 2026", "visibility": "unlisted" }
```

`200 OK` with the playlist.

```sql
UPDATE playlist SET name = :name, visibility = :visibility
WHERE playlist_id = :id AND owner_id = :me AND version = :if_match
RETURNING *;
-- 0 rows: 404 if it doesn't exist or isn't the caller's, otherwise 412 (stale version)
```

Changing to `unlisted` or `private` takes the playlist out of search and the owner's profile right away, because both filter on `visibility = 'public'`.

### `DELETE /playlists/{id}`

`204 No Content`. Owner only. Its entries, collaborators and follows are deleted with it (cascade).

### `GET /playlists/shared/{share_token}`

How unlisted playlists are opened: the share link is `https://app/p/<share_token>`. This returns the same body as `GET /playlists/{id}`, without `share_token`. An unknown token returns `404`.

```http
GET /api/v1/playlists/shared/ebe6a5ca-2615-46a8-99c5-4b97add2d265
→ 200 OK  "Boleros para mamá" (playlist 22, unlisted, owner diego.ramos)
```

The API should still refuse `private` playlists here: a share link only works while the playlist is `unlisted` or `public`.

### `POST /playlists/{id}/share-token` — new share link

Owner only. Old links stop working immediately.

```sql
UPDATE playlist SET share_token = gen_random_uuid()
WHERE playlist_id = :id AND owner_id = :me RETURNING share_token;
```

`200 OK` → `{ "share_token": "…" }`

### `POST /playlists/{id}/copy`

Copies a playlist the caller can view into a new playlist the caller owns. The body is optional, e.g. `{ "name": "My Road Trip" }`.

```sql
INSERT INTO playlist (name, description, owner_id, visibility)
SELECT coalesce(:name, name), description, :me, 'private' FROM playlist WHERE playlist_id = :id
RETURNING playlist_id;

INSERT INTO playlist_track (playlist_id, track_id)
SELECT :new_id, track_id FROM playlist_track WHERE playlist_id = :id ORDER BY position;
```

`201 Created` with the new playlist.

---

## Playlist tracks

A playlist holds **tracks** (specific recordings), not songs. An entry is identified by its **position**, because the same track can appear twice. To "delete a song from a playlist", either remove one entry by position, or remove every occurrence of a track with `?track_id=`.

Every write here needs `If-Match` and the owner or editor role. The database checks the role itself: adding as a non-editor fails with `insufficient_privilege`, which the API maps to 403.

### `GET /playlists/{id}/tracks`

```http
GET /api/v1/playlists/20/tracks?limit=3
```

```json
{
  "playlist_id": 20,
  "version": 1,
  "items": [
    {
      "position": 1,
      "added_at": "2026-11-05T20:43:11Z",
      "added_by": { "username": "ana.garcia", "display_name": "Ana García" },
      "track": {
        "track_id": 3674,
        "name": "Here Comes the Sun",
        "duration_ms": 185000,
        "duration": "3:05",
        "artists": ["The Beatles"],
        "album": { "album_id": 359, "title": "Abbey Road", "slug": "abbey-road", "artist_slug": "the-beatles" },
        "is_liked": true
      }
    },
    {
      "position": 2,
      "added_by": { "username": "ana.garcia", "display_name": "Ana García" },
      "track": { "track_id": 3668, "name": "Come Together", "duration": "4:20", "artists": ["The Beatles"], "album": { "album_id": 359, "title": "Abbey Road" } }
    },
    {
      "position": 3,
      "added_by": { "username": "ben.carter", "display_name": "Ben Carter" },
      "track": { "track_id": 4199, "name": "One More Time", "duration": "5:20", "artists": ["Daft Punk"], "album": { "title": "Discovery" } }
    }
  ],
  "limit": 3, "offset": 0, "total": 7
}
```

- **Database:** `playlist_track` joined to `track_detail` (for `track_artists`, album and duration) and `app_user` (for `added_by`), `ORDER BY position`.
- **`is_liked`:** only with a token, from `user_saved_track`.

### `POST /playlists/{id}/tracks` — add

Appends by default. With `position`, inserts there and pushes the following entries down.

```http
POST /api/v1/playlists/20/tracks
Authorization: Bearer eyJ...   (ben.carter, editor)
If-Match: "1"

{ "track_ids": [3735], "position": 3 }
```

```http
201 Created
ETag: "3"

{ "playlist_id": 20, "added": [ { "position": 3, "track_id": 3735, "added_by": "ben.carter" } ], "version": 3 }
```

```sql
-- only with a position: make room (the key is checked at the end of the statement)
UPDATE playlist_track SET position = position + :count
WHERE playlist_id = :id AND position >= :position;

INSERT INTO playlist_track (playlist_id, position, track_id, added_by)
SELECT :id, :position + n - 1, t.id, :me
FROM unnest(:track_ids::int[]) WITH ORDINALITY AS t(id, n);
-- without a position, leave it out: a trigger appends at the end
```

**Errors:**
- `403 forbidden`: the caller isn't the owner or an editor. Example: kenji.watanabe adding to *Road Trip* gets `user 7 can't add tracks to playlist 20: only its owner and editors can`.
- `412`: stale `If-Match`.
- `422`: an unknown track.

### `POST /playlists/{id}/tracks/move` — reorder

```http
POST /api/v1/playlists/20/tracks/move
If-Match: "1"

{ "from": 7, "to": 2 }
```

`200 OK` with the new version. "Around the World" moves from 7th to 2nd, and entries 2–6 shift down by one.

```sql
-- moving up (to < from); for moving down swap the bounds and use position - 1
UPDATE playlist_track
SET position = CASE WHEN position = :from THEN :to ELSE position + 1 END
WHERE playlist_id = :id AND position BETWEEN :to AND :from;
```

This runs as a single statement, so the temporary duplicate positions never fail the key check.

### `DELETE /playlists/{id}/tracks/{position}` — remove one entry

```http
DELETE /api/v1/playlists/20/tracks/1
If-Match: "1"
```

`200 OK` → `{ "removed": { "position": 1, "track_id": 3674 }, "version": 3 }`. The following entries move up one place.

```sql
WITH gone AS (
    DELETE FROM playlist_track WHERE playlist_id = :id AND position = :position RETURNING position, track_id
)
UPDATE playlist_track SET position = position - 1
WHERE playlist_id = :id AND position > (SELECT position FROM gone);
```

Check `playlist_can_edit(:id, :me)` first: the role trigger only guards inserts and changes of `added_by`.

### `DELETE /playlists/{id}/tracks?track_id=3674` — remove every occurrence

`200 OK` → `{ "removed": 1, "version": 3 }`. Delete the matching entries, then renumber all positions with `row_number()`, as section 2c of the seed does.

---

## Collaborators

### `GET /playlists/{id}/collaborators`

```json
{
  "items": [
    { "username": "ben.carter",  "display_name": "Ben Carter",  "role": "editor", "added_at": "2026-11-05T20:43:11Z" },
    { "username": "diego.ramos", "display_name": "Diego Ramos", "role": "editor", "added_at": "2026-11-05T20:43:11Z" }
  ]
}
```

Visible to the owner and the collaborators themselves.

### `PUT /playlists/{id}/collaborators/{username}` — add or change role

```http
PUT /api/v1/playlists/20/collaborators/kenji.watanabe
Authorization: Bearer eyJ...   (ana.garcia, owner)

{ "role": "editor" }
```

`200 OK` if the role changed, `201 Created` if they were added.

```sql
INSERT INTO playlist_collaborator (playlist_id, user_id, role)
VALUES (:id, (SELECT user_id FROM app_user WHERE username = :username), :role)
ON CONFLICT (playlist_id, user_id) DO UPDATE SET role = EXCLUDED.role;
```

**Errors:**
- `403`: the caller isn't the owner.
- `422`: the role isn't `editor` or `viewer`, or the user is the owner. For example, adding ana.garcia to her own playlist gives `user 2 owns playlist 20 and can't also be a collaborator`.

Making a private playlist visible to a friend: add them as a `viewer`.

### `DELETE /playlists/{id}/collaborators/{username}`

`204 No Content`. The owner can remove anyone, and a collaborator can remove themselves ("leave playlist"). Entries they added stay, still showing them as `added_by`.

---

## Library

`PUT` and `DELETE` here are idempotent: liking twice is fine and still returns `204`.

### `PUT /me/tracks/{track_id}` / `DELETE /me/tracks/{track_id}`

```sql
INSERT INTO user_saved_track (user_id, track_id) VALUES (:me, :track_id) ON CONFLICT DO NOTHING;
DELETE FROM user_saved_track WHERE user_id = :me AND track_id = :track_id;
```

`204 No Content`. An unknown track gives `404`.

### `GET /me/tracks`

Liked songs, newest first.

```http
GET /api/v1/me/tracks        (ben.carter)
```

```json
{
  "items": [
    { "saved_at": "2026-11-05T20:43:11Z", "track": { "track_id": 4249, "name": "Instant Crush", "artists": ["Daft Punk"], "album": { "title": "Random Access Memories" } } },
    { "saved_at": "2026-11-05T20:43:11Z", "track": { "track_id": 4201, "name": "Digital Love", "artists": ["Daft Punk"], "album": { "title": "Discovery" } } },
    { "saved_at": "2026-11-05T20:43:11Z", "track": { "track_id": 2233, "name": "Money", "artists": ["Pink Floyd"], "album": { "title": "The Dark Side of the Moon" } } }
  ],
  "limit": 20, "offset": 0, "total": 3
}
```

### `GET /me/tracks/contains?ids=3674,3668,4199`

Lets the app draw filled or empty hearts on a whole list at once.

```json
{ "3674": true, "3668": false, "4199": false }
```

```sql
SELECT t.id, EXISTS (SELECT 1 FROM user_saved_track s WHERE s.user_id = :me AND s.track_id = t.id)
FROM unnest(:ids::int[]) AS t(id);
```

### `GET /me/albums`, `PUT /me/albums/{album_id}`, `DELETE /me/albums/{album_id}`

These work the same way, on `user_saved_album`. Each listed item is an `album_summary` card.

---

## Follows

`PUT` follows and `DELETE` unfollows. Both are idempotent and return `204`. Follower counts update automatically through triggers, so the API never touches them.

| Endpoint | Table | Counter updated |
|---|---|---|
| `PUT /me/following/playlists/{id}` | `playlist_follow` | `playlist.follower_count` |
| `PUT /me/following/artists/{slug}` | `artist_follow` | `artist.follower_count` |
| `PUT /me/following/users/{username}` | `user_follow` | their `follower_count`, the caller's `following_count` |

```http
PUT /api/v1/me/following/artists/pink-floyd
→ 204 No Content        (Pink Floyd: follower_count 4 → 5)
```

**Errors:**
- `422` for following yourself (`user_follow_self_check`).
- `404` for a playlist the caller can't view.

The lists: `GET /me/following/artists`, `GET /me/following/playlists`, and `GET /playlists/{id}/followers`. That last one follows the same visibility rules as the playlist.

---

## Catalog

The catalog endpoints are read-only and public. The views do the joins, so most of these are a single query.

### `GET /artists`

Filters: `?type=group`, `?country=GB`, `?genre=rock`, `?sort=name|followers`. Reads `artist_summary`.

### `GET /artists/{slug}`

```http
GET /api/v1/artists/the-beatles
```

```json
{
  "artist_id": 276,
  "name": "The Beatles",
  "slug": "the-beatles",
  "artist_type": "group",
  "country_code": "GB",
  "city": "Liverpool",
  "formed_year": 1960,
  "disbanded_year": 1970,
  "biography": "English rock band formed in Liverpool in 1960. ...",
  "image_url": null,
  "follower_count": 3,
  "album_count": 20,
  "track_count": 462,
  "is_followed_by_me": false
}
```

### `GET /artists/{slug}/albums`

Every album the artist is on, including collaborations (`album_artist`), with the artist's role. Filters: `?type=studio|live|compilation|soundtrack`. Rows come from `album_summary`, ordered by `release_date NULLS LAST`.

```json
{
  "items": [
    { "album_id": 348, "title": "Please Please Me", "slug": "please-please-me", "release_date": "1963-03-22", "album_type": "studio", "role": "main", "track_count": 14 },
    { "album_id": 349, "title": "With the Beatles", "slug": "with-the-beatles", "release_date": "1963-11-22", "album_type": "studio", "role": "main", "track_count": 14 }
  ],
  "total": 20
}
```

### `GET /artists/{slug}/members`

The line-up from `artist_member` joined to `person`, one row per stint (Richard Wright appears twice for Pink Floyd: 1965–1979 and 1987–2008).

```json
{
  "items": [
    { "person_id": 1, "name": "John Lennon", "roles": ["vocals", "rhythm guitar", "keyboards", "harmonica"], "joined_year": 1960, "left_year": 1970 }
  ]
}
```

For a solo artist (`artist.person_id` set), answer with that one person instead.

### `GET /artists/{slug}/albums/{album_slug}` and `GET /albums/{id}`

```http
GET /api/v1/artists/the-beatles/albums/abbey-road
```

```json
{
  "album_id": 359,
  "title": "Abbey Road",
  "edition": null,
  "slug": "abbey-road",
  "release_date": "1969-09-26",
  "release_year": 1969,
  "album_type": "studio",
  "label": { "label_id": 4, "name": "Apple", "slug": "apple" },
  "artist": { "artist_id": 276, "name": "The Beatles", "slug": "the-beatles" },
  "artists": [ { "name": "The Beatles", "slug": "the-beatles", "role": "main" } ],
  "primary_genre": "Rock",
  "genres": ["Rock"],
  "track_count": 17,
  "disc_count": 1,
  "duration_ms": 2822000,
  "is_saved_by_me": true
}
```

- **Database:** `album_summary`, plus `album_artist` for `artists`. For a collaboration such as "Page & Plant", `artist` is the credited name and `artists` lists Jimmy Page and Robert Plant.
- **`is_saved_by_me`:** shown here for ana.garcia, who saved this album.

### `GET /albums/{id}/tracks`

From `track_detail WHERE album_id = :id ORDER BY disc_number, track_number`:

```json
{
  "items": [
    { "disc_number": 1, "track_number": 1, "track_id": 3668, "name": "Come Together", "duration": "4:20", "composer": "John Lennon, Paul McCartney", "song_id": 3281 },
    { "disc_number": 1, "track_number": 2, "track_id": 3669, "name": "Something", "duration": "3:03", "composer": "George Harrison", "song_id": 3282 },
    { "disc_number": 1, "track_number": 3, "track_id": 3670, "name": "Maxwell's Silver Hammer", "duration": "3:27", "composer": "John Lennon, Paul McCartney", "song_id": 3283 }
  ],
  "total": 17
}
```

### `GET /tracks/{id}`

One `track_detail` row: track, song, album, credited artist, `track_artists`, effective genre, composer, ISRC.

### `GET /songs/{id}` and `GET /songs/{id}/recordings`

The song (composition) with its writers from `credit`, and all its recordings:

```http
GET /api/v1/songs/3282/recordings
```

```json
{
  "song": { "song_id": 3282, "title": "Something", "first_release_year": 1969,
            "credits": [ { "person_id": 3, "name": "George Harrison", "role": "writer" } ] },
  "items": [
    { "track_id": 3669, "album_title": "Abbey Road",  "release_year": 1969, "duration": "3:03" },
    { "track_id": 3746, "album_title": "1967-1970",   "release_year": 1973, "duration": "3:00" },
    { "track_id": 3931, "album_title": "Anthology 3", "release_year": 1996, "duration": "3:18" },
    { "track_id": 3962, "album_title": "1",           "release_year": 2000, "duration": "3:02" }
  ]
}
```

Each credit shows `credited_as` when it's set (Ringo Starr's songs show "Richard Starkey").

### `GET /people/{id}`

A person page: the person's details, the bands they were in (`artist_member`), the solo artist they are (`artist.person_id`), and their songwriting credits (`credit` → `song`). Examples 5 and 8 in [examples.sql](../db/queries/examples.sql).

### `GET /genres` and `GET /genres/{id}/albums`

`/genres` returns the tree, nested through `parent_genre_id`:

```json
[ { "genre_id": 1, "name": "Rock", "children": [ { "name": "Rock And Roll" }, { "name": "Progressive Rock" }, { "name": "Psychedelic Rock" } ] } ]
```

`/genres/{id}/albums` includes subgenres (a recursive CTE, example 10). Add `?primary_only=true` to match only albums whose *primary* genre is in the tree.

### `GET /labels/{slug}`

The label, with its albums (`album_summary WHERE label_id = :id`).

---

## Search

### `GET /search?q=come together&types=song,track&limit=5`

One call for the search box. It wraps `search_catalog(q, limit)`, which matches accent-insensitive substrings and full text across artists, albums, songs, tracks, people, **public** playlists and users.

```json
{
  "query": "come together",
  "items": [
    { "kind": "song",  "id": 3281, "name": "Come Together", "context": "The Beatles", "score": 1.0 },
    { "kind": "track", "id": 3932, "name": "Come Together", "context": "Anthology 3 — The Beatles", "score": 1.0 },
    { "kind": "track", "id": 3963, "name": "Come Together", "context": "1 — The Beatles", "score": 1.0 }
  ]
}
```

- **`types`:** filters on `kind` (`artist`, `album`, `song`, `track`, `person`, `playlist`, `user`).
- **`slug`:** included where the kind has one, for building links (artists, albums, and usernames for users).
- **Accents:** `?q=jose` finds "José José".

---

## Reference data

### `GET /reference/{name}`

The allowed values for dropdowns and validation, straight from the lookup tables. `name` is one of `countries`, `artist-types`, `album-types`, `artist-roles`, `credit-roles`, `playlist-visibilities` or `playlist-roles`.

```http
GET /api/v1/reference/playlist-visibilities
```

```json
[
  { "code": "public",   "description": "Listed on the owner's profile and in search" },
  { "code": "unlisted", "description": "Not listed; anyone with the share link can open it" },
  { "code": "private",  "description": "Only the owner and collaborators" }
]
```

These rarely change, so serve them with a long `Cache-Control`.

---

## Catalog administration

Writes to the catalog (artists, albums, tracks, credits) need an **admin role, which the schema doesn't have yet**. `is_verified` marks editorial and artist accounts, not permissions. Before building these endpoints, add something like `app_user.role` (`listener`, `curator`, `admin`). Alternatively, load catalog data with SQL, the way the seed does.

| Method | Path | Notes |
|---|---|---|
| `POST` | `/admin/artists` | Slug is generated when omitted |
| `PATCH` | `/admin/artists/{id}` | The slug doesn't change with the name; send `slug` to change it |
| `POST` | `/admin/albums` | Must include its artists. See below |
| `POST` | `/admin/albums/{id}/tracks` | Tracks with disc/track number, optional `genre_id` (added to the album's genres automatically) |
| `POST` | `/admin/songs/{id}/credits` | `track.composer` of every recording regenerates automatically |
| `PUT` | `/admin/albums/{id}/genres` | Exactly one `is_primary` |

Some rules are checked at **commit**, not per statement. So an endpoint that creates an album must insert the album and its `album_artist` rows in **one transaction**:

```json
POST /api/v1/admin/albums
{
  "title": "Let It Be... Naked",
  "artist_id": 276,
  "album_type": "studio",
  "release_date": "2003-11-17",
  "artists": [ { "artist_id": 276, "role": "main" } ],
  "genres":  [ { "genre_id": 1, "is_primary": true } ]
}
```

If the artists or the primary genre are missing, the commit fails with `integrity_constraint_violation`, which the API returns as `422`.

---

## Mapping database errors to HTTP

The database enforces the rules, so the API can translate its errors (by SQLSTATE) instead of repeating the checks:

| SQLSTATE | Name | Raised by | HTTP |
|---|---|---|---|
| `23505` | unique_violation | duplicate username, email, slug, ISRC… | `409 Conflict` |
| `23503` | foreign_key_violation | unknown track, album, user, country or lookup code | `422` (body) or `404` (URL) |
| `23514` | check_violation | bad username format, following yourself, negative duration… | `422` |
| `23P01` | exclusion_violation | overlapping band membership stints | `422` |
| `23000` | integrity_constraint_violation | album without its artists, wrong number of primary genres, owner as collaborator | `422` |
| `42501` | insufficient_privilege | adding to a playlist without being the owner or an editor | `403 Forbidden` |
| *0 rows updated* | — | `WHERE version = :if_match` didn't match | `412 Precondition Failed` (or `404` if the row doesn't exist) |

The constraint name in the error (for example `app_user_username_key`) tells the API which field to blame in the response.
