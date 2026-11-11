# DB Design

The database changes behind the web UI ([UI-Design.md](UI-Design.md)) and the API changes it needs ([API-Design.md](API-Design.md), sections marked **(UI)**). This file holds the SQL for each change, so the changes can be moved into the database project (`db-postgresql-music`) later and fitted to its structure. The API documents only behavior, and links here for the SQL.

**Status:** proposal. Most of these changes are already in the database project (`db-postgresql-music`), with DB-04 at 100 tracks. DB-04's new limit of **20 tracks** (2026-11-11) is made in its working tree but not committed, and the running database still has 100 until `make reset`. Each change has an ID (**DB-xx**). The API spec, the UI spec and future migrations can refer to that ID.

**Contents**

- [How to use this file](#how-to-use-this-file)
- [Summary of changes](#summary-of-changes)
- [Assumptions about the current schema](#assumptions-about-the-current-schema)
- [DB-00 Prerequisites](#db-00-prerequisites)
- [DB-01 Favorite limit (5 per user)](#db-01-favorite-limit-5-per-user)
- [DB-02 Favoriting your own playlist](#db-02-favoriting-your-own-playlist)
- [DB-03 Visibility follows favorites](#db-03-visibility-follows-favorites)
- [DB-04 Track limit (20 per playlist)](#db-04-track-limit-20-per-playlist)
- [DB-05 Playlist limit (20 per owner)](#db-05-playlist-limit-20-per-owner)
- [DB-06 Random tracks](#db-06-random-tracks)
- [DB-07 Random playlists](#db-07-random-playlists)
- [DB-08 Song search](#db-08-song-search)
- [DB-09 Indexes for "my playlists"](#db-09-indexes-for-my-playlists)
- [DB-10 Auth schema for JWT login (API-owned)](#db-10-auth-schema-for-jwt-login-api-owned)
- [Error mapping](#error-mapping)
- [Migration order and rollback](#migration-order-and-rollback)
- [Verification script](#verification-script)
- [Open items](#open-items)

---

## How to use this file

1. **Check the assumptions first.** The SQL uses the table, column and view names that API-Design.md uses. The [Assumptions](#assumptions-about-the-current-schema) table lists the ones the API spec doesn't fully confirm. Check each against the database project's schema and data model (`data-model.md`) before moving anything.
2. **One change, one migration.** Each DB-xx section is self-contained: its objects, its SQL, how to undo it, and how to check it. When the changes move to the database project, each becomes one migration file (or one section of its schema file), in the [order](#migration-order-and-rollback) given below.
3. **Keep names.** Constraint and trigger names (`playlist_follow_limit`, `playlist_track_limit`…) are part of the contract: the API maps errors to HTTP by constraint name (see [Error mapping](#error-mapping)). Renaming one means changing the API's mapping too.
4. **Then update the docs.** After a change moves, mark it as done here, and update the database project's `data-model.md` and `make verify` counts.

---

## Summary of changes

| ID | Change | Kind | Why (UI rule / API gap) | Used by (API) |
|---|---|---|---|---|
| DB-00 | `unaccent` and `pg_trgm` extensions; `immutable_unaccent()` | prerequisite | – | DB-08 |
| DB-01 | At most **5 favorites** per user | trigger | BR-04 / G-01 | `PUT /me/following/playlists/{id}` |
| DB-02 | Owners can favorite their own playlist | check only | BR-05 / G-01b | same |
| DB-03 | **Visibility follows favorites**: `public` while any favorite exists, `private` otherwise; nobody else can change it; new playlists are `private` | triggers, default, data migration | BR-10, BR-12, BR-13, D-10, D-12, D-13 / G-10, G-11 | `POST`, `PATCH /playlists`, follows |
| DB-04 | At most **20 tracks** per playlist (was 100) | trigger | BR-03 / G-03 | `POST /playlists`, `POST /playlists/{id}/tracks`, `/copy` |
| DB-05 | At most **20 owned playlists** per user | trigger | BR-02 / G-04 | `POST /playlists`, `/copy` |
| DB-06 | `random_tracks(limit, exclude)` | function | BR-08 / G-05 | `GET /tracks/random` |
| DB-07 | `random_playlists(user, limit, exclude)` + partial index | function, index | BR-07 / G-06 | `GET /playlists/random` |
| DB-08 | `track_search` materialized view + `search_tracks(q, limit, offset)` | view, function, indexes | D-08 / G-12 | `GET /tracks?q=` |
| DB-09 | Indexes for `last_added_at` and `contains_track` | indexes | BR-09, BR-11 / G-07, G-08 | `GET /me/playlists` |
| DB-10 | `auth` schema: credentials, roles, refresh tokens (JWT login) | reference: owned by the API | D-01, D-11 | `/auth/*`, `POST /users` |

---

## Assumptions about the current schema

Names taken from API-Design.md. **Verify each one** against the database project before using the SQL.

| Object | Assumed | Used by | Source in API-Design.md |
|---|---|---|---|
| `app_user` | `user_id int` PK, `username` | DB-01, DB-05 | Account |
| `playlist` | `playlist_id int` PK, `owner_id` → `app_user`, `name`, `visibility` → lookup of visibilities, `follower_count`, `version`, `share_token` | DB-03 to DB-05, DB-07 | Playlists |
| `playlist_track` | `(playlist_id, position)` key, `track_id`, `added_by` (nullable), `added_at` | DB-04, DB-09 | Playlist tracks |
| `playlist_follow` | `(user_id, playlist_id)` key, a timestamp (assumed `followed_at`), `ON DELETE CASCADE` from `playlist` and `app_user` | DB-01 to DB-03 | Follows |
| Follower counter | A trigger on `playlist_follow` keeps `playlist.follower_count` | DB-03 (must not conflict) | Follows |
| `playlist_summary` | A view with `playlist_id`, `owner_id`, `visibility`, … | DB-07 | Playlists |
| `track` | `track_id int` PK, dense IDs (few gaps) | DB-06 | Catalog |
| `track_detail` | A view with `track_id`, `name`, `album_title`, `track_artists text[]` | DB-08 | Catalog |
| Visibility values | `public`, `unlisted`, `private` in a lookup table | DB-03 | Reference data |
| `search_catalog(q, limit)` | Exists; accent-insensitive. DB-08 copies its approach for tracks. | DB-08 | Search |

Two checks matter most:
- **DB-02:** `playlist_follow` must have **no** check that stops an owner from following their own playlist.
- **DB-08:** the real column names in `track_detail` for the album title and the artist names.

---

## DB-00 Prerequisites

```sql
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- unaccent() is STABLE, so it can't be used in an index. This wrapper is safe to mark IMMUTABLE
-- because it names the dictionary explicitly. Reuse the database's own wrapper if it already has one.
CREATE OR REPLACE FUNCTION immutable_unaccent(text) RETURNS text
LANGUAGE sql IMMUTABLE PARALLEL SAFE STRICT
AS $$ SELECT public.unaccent('public.unaccent'::regdictionary, $1) $$;
```

`search_catalog` is already accent-insensitive, so `unaccent` probably exists. `IF NOT EXISTS` makes this safe either way.

---

## DB-01 Favorite limit (5 per user)

A favorite is a `playlist_follow` row. A user has at most 5, counting their own playlists and other people's.

```sql
CREATE OR REPLACE FUNCTION playlist_follow_limit() RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  -- Lock the user's row so two favorites added at the same moment are counted one after the other.
  PERFORM 1 FROM app_user WHERE user_id = NEW.user_id FOR UPDATE;
  IF (SELECT count(*) FROM playlist_follow WHERE user_id = NEW.user_id) >= 5 THEN
    RAISE EXCEPTION 'user % already has 5 favorite playlists', NEW.user_id
      USING ERRCODE = 'check_violation', CONSTRAINT = 'playlist_follow_limit';
  END IF;
  RETURN NEW;
END $$;

CREATE TRIGGER playlist_follow_limit
  BEFORE INSERT ON playlist_follow
  FOR EACH ROW EXECUTE FUNCTION playlist_follow_limit();
```

- **Swap (the UI's Replace dialog):** the API deletes the old row and inserts the new one in one transaction. The delete runs first, so the count is 4 when the insert is checked.
- **Idempotent favorite:** the API inserts with `ON CONFLICT DO NOTHING`. A conflicting row is never inserted, but the `BEFORE INSERT` trigger still runs first. So a user with 5 favorites who favorites one of them again would get the error. The API must check `EXISTS` first and answer `200` without inserting.
- **Existing data:** before creating the trigger, find users who already have more than 5:
  ```sql
  SELECT user_id, count(*) FROM playlist_follow GROUP BY user_id HAVING count(*) > 5;
  ```
  The seed should have none. If any exist, decide which follows to keep before the migration runs.
- **Undo:** `DROP TRIGGER playlist_follow_limit ON playlist_follow; DROP FUNCTION playlist_follow_limit();`

---

## DB-02 Favoriting your own playlist

This needs no new SQL, only a check that nothing forbids it:

```sql
SELECT conname, pg_get_constraintdef(oid)
FROM pg_constraint WHERE conrelid = 'playlist_follow'::regclass;

SELECT tgname FROM pg_trigger WHERE tgrelid = 'playlist_follow'::regclass AND NOT tgisinternal;
```

If a check or trigger compares `user_id` with the playlist's `owner_id`, remove it: favoriting your own playlist is how it becomes public (DB-03). `user_follow_self_check` on `user_follow` is a different table and stays.

---

## DB-03 Visibility follows favorites

The rule (UI decisions D-06, D-10, D-12, D-13):

| The playlist has… | `visibility` |
|---|---|
| at least one `playlist_follow` row (its owner's or anyone's) | `public` |
| none | `private` |

What this means in practice:
- New playlists are `private`.
- The owner unfavoriting their playlist leaves it `public` while others have it.
- Only the last favorite going away makes it `private`.
- The owner can't make a favorited playlist private, except by deleting it.

Discover and search list it for as long as it's `public`.

Four pieces:

**1. The default.** New playlists start private.

```sql
ALTER TABLE playlist ALTER COLUMN visibility SET DEFAULT 'private';
```

**2. Keep `visibility` in step with favorites.**

```sql
CREATE OR REPLACE FUNCTION playlist_visibility_from_favorites() RETURNS trigger
LANGUAGE plpgsql AS $$
DECLARE
  pid    int := coalesce(NEW.playlist_id, OLD.playlist_id);
  wanted text;
BEGIN
  -- Lock the playlist so two favorites removed at the same moment see each other's change.
  -- (In READ COMMITTED each statement below takes a new snapshot after the lock is granted.)
  PERFORM 1 FROM playlist WHERE playlist_id = pid FOR UPDATE;
  IF NOT FOUND THEN
    RETURN NULL;               -- the playlist itself is being deleted (cascade): nothing to do
  END IF;

  wanted := CASE WHEN EXISTS (SELECT 1 FROM playlist_follow WHERE playlist_id = pid)
                 THEN 'public' ELSE 'private' END;

  UPDATE playlist SET visibility = wanted
  WHERE playlist_id = pid AND visibility IS DISTINCT FROM wanted;
  RETURN NULL;
END $$;

CREATE TRIGGER playlist_visibility_from_favorites
  AFTER INSERT OR DELETE ON playlist_follow
  FOR EACH ROW EXECUTE FUNCTION playlist_visibility_from_favorites();
```

**3. Nobody else changes it.** A guard on `playlist` allows visibility changes only from inside a trigger (the one above), and makes every new playlist private:

```sql
CREATE OR REPLACE FUNCTION playlist_visibility_guard() RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    NEW.visibility := 'private';                     -- a new playlist has no favorites yet
  ELSIF NEW.visibility IS DISTINCT FROM OLD.visibility AND pg_trigger_depth() < 2 THEN
    RAISE EXCEPTION 'playlist %: visibility follows favorites and can''t be set directly', NEW.playlist_id
      USING ERRCODE = 'check_violation', CONSTRAINT = 'playlist_visibility_derived';
  END IF;
  RETURN NEW;
END $$;

CREATE TRIGGER playlist_visibility_guard
  BEFORE INSERT OR UPDATE OF visibility ON playlist
  FOR EACH ROW EXECUTE FUNCTION playlist_visibility_guard();
```

`pg_trigger_depth()` is 1 for a statement a client sends and 2 for the `UPDATE` that the trigger in step 2 runs. The API also refuses `visibility` in requests (`422 visibility_follows_favorites`), so the guard is a safety net, not the main check.

**4. Migrate existing data.** Run this **before** creating the guard, because the guard would refuse it:

```sql
UPDATE playlist p
SET visibility = CASE WHEN EXISTS (SELECT 1 FROM playlist_follow f WHERE f.playlist_id = p.playlist_id)
                      THEN 'public' ELSE 'private' END;
```

What happens to the seed:
- Playlists without follows become private.
- Unlisted ones become public or private depending on their follows. For example, *Boleros para mamá* (playlist 22) is unlisted in the seed.
- API-Design.md's share-link example then no longer applies to the seed. Share tokens stay in place for a future "share a private playlist" feature.

The seed itself also needs updating: it should insert playlists without a visibility, and let its follows decide.

**Interactions to check:**
- **The follower counter:** its trigger also updates `playlist` (`follower_count`), not `visibility`, so the guard lets it through. Both triggers lock the same row, so they don't deadlock with each other.
- **`playlist.version`:** a visibility change doesn't touch `version`, which counts track-list changes only.
- **`updated_at`:** the trigger that sets `updated_at` will mark a visibility change as an update. That's correct.
- **`DELETE /me`:** a user's deletion cascades to their follows. If they were the last favorite of someone else's playlist, that playlist becomes private. Their own playlists are deleted.
- **Isolation level:** the lock in step 2 is enough under `READ COMMITTED`, PostgreSQL's default. Under `REPEATABLE READ` or `SERIALIZABLE`, a concurrent change gives a serialization error, which the API should retry.

**Undo:** drop both triggers and both functions; put the old default back (`SET DEFAULT 'public'`). The data can't be undone automatically: keep a copy of `(playlist_id, visibility)` from before the migration if you might need it.

---

## DB-04 Track limit (20 per playlist)

```sql
CREATE OR REPLACE FUNCTION playlist_track_limit() RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  PERFORM 1 FROM playlist WHERE playlist_id = NEW.playlist_id FOR UPDATE;   -- one add at a time per playlist
  IF (SELECT count(*) FROM playlist_track WHERE playlist_id = NEW.playlist_id) >= 20 THEN
    RAISE EXCEPTION 'playlist % already has 20 tracks', NEW.playlist_id
      USING ERRCODE = 'check_violation', CONSTRAINT = 'playlist_track_limit';
  END IF;
  RETURN NEW;
END $$;

CREATE TRIGGER playlist_track_limit
  BEFORE INSERT ON playlist_track
  FOR EACH ROW EXECUTE FUNCTION playlist_track_limit();
```

- **What it covers:** it runs per row, so a multi-row insert (`POST /playlists` with `track_ids`, `/copy`, adding several tracks) stops at the 21st row, and the whole statement rolls back. The API checks the count first so it can say how many still fit.
- **Name order:** if the database already has a `BEFORE INSERT` trigger on `playlist_track` (the one that fills in `position` on append), the order doesn't matter: that one sets columns and this one only counts rows. PostgreSQL runs triggers of the same kind in name order.
- **Moves and removals** never insert rows, so they're not affected.
- **Existing data:** the trigger only checks inserts, so a playlist that already holds more than 20 tracks keeps them. Nothing more can be added to it, but its tracks can still be moved and removed. The seed has nine such playlists, all owned by `musicdb`. This query lists them: `SELECT playlist_id, count(*) FROM playlist_track GROUP BY 1 HAVING count(*) > 20;`

  | Playlist | ID | Tracks |
  |---|---|---|
  | *Music* | 1 | 3289 |
  | *90's Music* | 5 | 1477 |
  | *The Beatles: Original Albums* | 19 | 193 |
  | *Classical* | 12 | 75 |
  | *Brazilian Music* | 11 | 39 |
  | *Heavy Metal Classic* | 17 | 26 |
  | *Classical 101 - Deep Cuts*, *- Next Steps*, *- The Basics* | 13, 14, 15 | 25 each |

  They stay as they are, like the first three, which were already over 100. To make the seed obey the limit, trim them instead (DBO-07).
- **Changed 2026-11-11 (100 → 20):** a database that already has the 100-track trigger only needs the `CREATE OR REPLACE FUNCTION` above: the trigger and its name stay the same. Also update the seed's list of playlists that are older than the limit (`02_data.sql`), the `make verify` rule (*at most 20 tracks per playlist, 9 sample playlists excepted*), and `data-model.md`. To undo this change, run the function again with 100.
- **Undo:** drop the trigger and the function.

---

## DB-05 Playlist limit (20 per owner)

```sql
CREATE OR REPLACE FUNCTION playlist_owner_limit() RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  PERFORM 1 FROM app_user WHERE user_id = NEW.owner_id FOR UPDATE;
  IF (SELECT count(*) FROM playlist WHERE owner_id = NEW.owner_id) >= 20 THEN
    RAISE EXCEPTION 'user % already owns 20 playlists', NEW.owner_id
      USING ERRCODE = 'check_violation', CONSTRAINT = 'playlist_owner_limit';
  END IF;
  RETURN NEW;
END $$;

CREATE TRIGGER playlist_owner_limit
  BEFORE INSERT ON playlist
  FOR EACH ROW EXECUTE FUNCTION playlist_owner_limit();
```

- **Covers** `POST /playlists` and `POST /playlists/{id}/copy`. Playlists don't change owner, so `UPDATE` isn't covered. If ownership transfer is ever added, add `OR UPDATE OF owner_id`.
- **Counts owned playlists only.** Collaborations don't count (UI-Design.md Q-02 default).
- **Existing data:** `SELECT owner_id, count(*) FROM playlist GROUP BY 1 HAVING count(*) > 20;` should return nothing.
- **Undo:** drop the trigger and the function.

---

## DB-06 Random tracks

`ORDER BY random()` over `track` reads the whole table, and the catalog has no size limit. Picking random IDs uses the primary key index instead:

```sql
CREATE OR REPLACE FUNCTION random_tracks(p_limit int DEFAULT 20, p_exclude int[] DEFAULT '{}')
RETURNS SETOF int
LANGUAGE sql VOLATILE AS $$
  WITH bounds AS (SELECT min(track_id) AS lo, max(track_id) AS hi FROM track),
  picks AS (
    SELECT DISTINCT (lo + floor(random() * (hi - lo + 1)))::int AS track_id
    FROM bounds, generate_series(1, greatest(p_limit, 1) * 3)       -- 3× to make up for gaps and exclusions
  )
  SELECT t.track_id
  FROM picks JOIN track t USING (track_id)
  WHERE t.track_id <> ALL (p_exclude)
  LIMIT p_limit
$$;
```

- **Use:** `SELECT td.* FROM random_tracks(20, '{3674,3668}') r(track_id) JOIN track_detail td USING (track_id);`
- **Fair** while track IDs are dense. If the catalog ever has big gaps, the result can come back short: the API then calls again, or switches to `TABLESAMPLE SYSTEM_ROWS` (extension `tsm_system_rows`). That one samples whole disk pages, so its results cluster by album.
- **Undo:** `DROP FUNCTION random_tracks(int, int[]);`

---

## DB-07 Random playlists

Public playlists by other users that the caller hasn't favorited. That includes playlists kept public only by other people's favorites (D-12).

```sql
CREATE OR REPLACE FUNCTION random_playlists(p_user int, p_limit int DEFAULT 12, p_exclude int[] DEFAULT '{}')
RETURNS SETOF int
LANGUAGE sql VOLATILE AS $$
  SELECT p.playlist_id
  FROM playlist p
  WHERE p.visibility = 'public'
    AND p.owner_id IS DISTINCT FROM p_user
    AND p.playlist_id <> ALL (p_exclude)
    AND (p_user IS NULL OR NOT EXISTS (
          SELECT 1 FROM playlist_follow f WHERE f.playlist_id = p.playlist_id AND f.user_id = p_user))
  ORDER BY random()
  LIMIT p_limit
$$;

CREATE INDEX IF NOT EXISTS playlist_public_idx ON playlist (playlist_id) WHERE visibility = 'public';
```

- **Use:** `SELECT ps.* FROM random_playlists(:me, 12, :exclude) r(playlist_id) JOIN playlist_summary ps USING (playlist_id);` With no token, `p_user` is `NULL`.
- **Cost:** every public playlist has at least one favorite, and each user has at most 5. So there are never more than 5 × the number of users, and `ORDER BY random()` over that set stays cheap for a long time.
- **Undo:** drop the function and the index.

---

## DB-08 Song search

The Songs screen searches track names, artist names and album titles, ignoring accents and case (`jose` finds *José José*). `search_catalog` only matches track names for tracks, so tracks get their own search, built on a materialized view:

```sql
CREATE MATERIALIZED VIEW track_search AS
SELECT td.track_id,
       immutable_unaccent(lower(td.name)) AS name_norm,
       immutable_unaccent(lower(concat_ws(' ', td.name,
                                          array_to_string(td.track_artists, ' '),
                                          td.album_title))) AS search_text
FROM track_detail td;

CREATE UNIQUE INDEX track_search_pk   ON track_search (track_id);           -- needed for REFRESH … CONCURRENTLY
CREATE INDEX        track_search_trgm ON track_search USING gin (search_text gin_trgm_ops);
CREATE INDEX        track_search_fts  ON track_search USING gin (to_tsvector('simple', search_text));

CREATE OR REPLACE FUNCTION search_tracks(p_q text, p_limit int DEFAULT 20, p_offset int DEFAULT 0)
RETURNS TABLE (track_id int, score real, total bigint)
LANGUAGE sql STABLE AS $$
  WITH q AS (SELECT immutable_unaccent(lower(btrim(p_q))) AS q),
  hits AS (
    SELECT s.track_id, s.name_norm,
           (CASE WHEN s.name_norm = q.q              THEN 1.0
                 WHEN s.name_norm LIKE q.q || '%'    THEN 0.9
                 WHEN s.name_norm LIKE '%' || q.q || '%' THEN 0.8
                 ELSE 0.5 + 0.3 * word_similarity(q.q, s.search_text)
            END)::real AS score
    FROM track_search s, q
    WHERE length(q.q) >= 2
      AND (s.search_text LIKE '%' || q.q || '%'
           OR to_tsvector('simple', s.search_text) @@ plainto_tsquery('simple', q.q))
  )
  SELECT track_id, score, count(*) OVER () AS total
  FROM hits
  ORDER BY score DESC, name_norm, track_id
  LIMIT p_limit OFFSET p_offset
$$;
```

- **Use:** `SELECT td.*, s.score, s.total FROM search_tracks('daft punk', 20, 0) s JOIN track_detail td USING (track_id) ORDER BY s.score DESC, td.name;`
- **Matching:**
  - A substring anywhere in title, artist or album matches: `punk` finds every Daft Punk track. The trigram index serves `LIKE '%…%'`.
  - Words in any order also match through full text: `punk daft`.
  - Matches in the track's own name rank first.
- **Total:** `total` is the same on every row. The API returns it as the page's `total` ("9 songs match").
- **Fewer than 2 characters** returns nothing. The API answers `422` before calling.
- **Refresh:** the catalog changes only through catalog loads and `/admin` writes. After each one, run `REFRESH MATERIALIZED VIEW CONCURRENTLY track_search;`. The database project's load script should do it at the end; the API's catalog-admin endpoints after a write. If catalog writes become frequent, replace the view with a table kept current by triggers.
- **Check the column names** `name`, `album_title`, `track_artists` against `track_detail` ([Assumptions](#assumptions-about-the-current-schema)).
- **Undo:** `DROP FUNCTION search_tracks(text, int, int); DROP MATERIALIZED VIEW track_search;`

---

## DB-09 Indexes for "my playlists"

`GET /me/playlists` gains `last_added_at` (for the add menu's "Last used") and `contains_track` (for "✓ Added"):

```sql
-- last_added_at: max(added_at) WHERE playlist_id = … AND added_by = :me
CREATE INDEX IF NOT EXISTS playlist_track_added_by_idx ON playlist_track (added_by, playlist_id, added_at DESC);

-- contains_track: EXISTS (… WHERE playlist_id = … AND track_id = :track)
CREATE INDEX IF NOT EXISTS playlist_track_track_idx ON playlist_track (track_id, playlist_id);
```

The query the API runs, for reference:

```sql
SELECT ps.*,
       la.last_added_at,
       EXISTS (SELECT 1 FROM playlist_track pt
               WHERE pt.playlist_id = ps.playlist_id AND pt.track_id = :contains_track) AS contains_track,
       EXISTS (SELECT 1 FROM playlist_follow f
               WHERE f.playlist_id = ps.playlist_id AND f.user_id = :me)               AS is_favorite
FROM playlist_summary ps
LEFT JOIN LATERAL (SELECT max(added_at) AS last_added_at FROM playlist_track
                   WHERE playlist_id = ps.playlist_id AND added_by = :me) la ON true
WHERE ps.owner_id = :me                                   -- ?role=owner
ORDER BY immutable_unaccent(lower(ps.name));              -- ?sort=name
```

- The foreign key on `playlist_track.track_id` may already have an index. If so, skip the second one.
- **Undo:** drop the indexes.

---

## DB-10 Auth schema for JWT login (API-owned)

Login uses **JWT** (UI decision D-11). This section is a **reference**: the `auth` schema belongs to the API (`api-spring-music`), which creates it with its own Flyway migrations, and the music schema (`public`) never holds passwords. It's listed here so the database project knows the schema exists and doesn't touch it.

| What | Where | Notes |
|---|---|---|
| **Access token** (JWT) | **Not stored.** Signed by the API with HMAC-SHA256 (`HS256`), valid 15 minutes. | Claims: `iss`, `aud`, `sub` (= `app_user.auth_subject`), `iat`, `nbf`, `exp`, `jti`, `preferred_username`, `role`. The signing secret is API configuration, never in the database. |
| **Credentials** | `auth.user_account`: one row per `app_user`, with the bcrypt password hash (delegating `{id}` prefix) and the role (`listener`, `curator`, `admin` → `auth.role`). | Deleting the `app_user` deletes it (cascade). An account without a password (the seed users) can't log in. |
| **Refresh tokens** | `auth.refresh_token`: opaque, 32 random bytes, stored **only as a SHA-256 hash**, single-use, 14 days. Each row has a token family (one per login), its user, issue and expiry times, and when it was used or revoked. | Reusing a used token revokes its family. A daily job deletes rows that expired more than a day ago. |

Shape, for orientation only. The API's migrations are the source of truth:

```sql
CREATE SCHEMA auth;
CREATE TABLE auth.role (code text PRIMARY KEY);                 -- listener, curator, admin
CREATE TABLE auth.user_account (
  user_id       int  PRIMARY KEY REFERENCES public.app_user ON DELETE CASCADE,
  password_hash text,                                           -- null: can't log in
  role          text NOT NULL DEFAULT 'listener' REFERENCES auth.role,
  updated_at    timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE auth.refresh_token (
  refresh_token_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id    int         NOT NULL REFERENCES public.app_user ON DELETE CASCADE,
  family_id  uuid        NOT NULL,
  token_hash bytea       NOT NULL UNIQUE,                       -- sha256(token)
  issued_at  timestamptz NOT NULL DEFAULT now(),
  expires_at timestamptz NOT NULL,
  used_at    timestamptz,
  revoked_at timestamptz
);
CREATE INDEX ON auth.refresh_token (family_id);
CREATE INDEX ON auth.refresh_token (expires_at);
```

For the database project: `make verify` counts `app_user` rows, and the API adds the super user (`oscargm`), which changes the count (REQUIREMENTS.md §24).

---

## Error mapping

The API translates these database errors by SQLSTATE and constraint name. These rows extend [Mapping database errors to HTTP](API-Design.md#mapping-database-errors-to-http).

| Raised by | SQLSTATE | Constraint name | HTTP | Error code |
|---|---|---|---|---|
| DB-01 | `23514` check_violation | `playlist_follow_limit` | 422 | `favorite_limit_reached` |
| DB-03 | `23514` check_violation | `playlist_visibility_derived` | 422 | `visibility_follows_favorites` |
| DB-04 | `23514` check_violation | `playlist_track_limit` | 422 | `playlist_full` |
| DB-05 | `23514` check_violation | `playlist_owner_limit` | 422 | `playlist_limit_reached` |
| DB-03 (with `REPEATABLE READ` or `SERIALIZABLE`) | `40001` serialization_failure | – | retry once, then 409 | `conflict` |

`RAISE … USING CONSTRAINT = '…'` fills in the error's constraint field, the same field that real constraints fill in, so the API's existing "map by constraint name" code handles these too.

---

## Migration order and rollback

Run in this order. Each step is safe to run alone.

| Step | Change | Notes |
|---|---|---|
| 1 | DB-00 | Extensions and `immutable_unaccent`. |
| 2 | DB-02 | Check only. Fix any self-check before going on. |
| 3 | Data checks for DB-01, DB-04, DB-05 | The three "more than the limit" queries must return nothing, except the nine seed playlists listed in DB-04. |
| 4 | DB-01, DB-04, DB-05 | The three limit triggers. Independent of each other. |
| 5 | DB-03 part 1 and 4 | New default, then migrate the data. |
| 6 | DB-03 part 2 and 3 | The sync trigger, then the guard. The guard must come **after** the data migration. |
| 7 | DB-06, DB-07 | Functions and index. |
| 8 | DB-08 | View, indexes, function. The first build reads all of `track_detail`. |
| 9 | DB-09 | Indexes. Use `CREATE INDEX CONCURRENTLY` on a live database (outside a transaction). |

**Rollback:** reverse order, using each section's **Undo**. The only change that can't be undone automatically is DB-03's data migration; save `(playlist_id, visibility)` before step 5.

**The API must ship with or after these changes.** The API's new checks (refusing `visibility`, the follow-response body) work without them, but the limits and the visibility rule only hold once the triggers exist.

---

## Verification script

Run after the migration, inside a transaction that's rolled back at the end. It uses seed users and playlists, so the IDs come from lookups. Each check shows the result to expect.

```sql
BEGIN;

-- Helpers
CREATE TEMP TABLE u AS SELECT username, user_id FROM app_user
  WHERE username IN ('ana.garcia','ben.carter','kenji.watanabe');

-- DB-05/DB-03: a new playlist is private, whatever the client sends
INSERT INTO playlist (name, owner_id, visibility)
VALUES ('Verify', (SELECT user_id FROM u WHERE username='ana.garcia'), 'public')
RETURNING playlist_id, visibility;                                   -- expect: private

-- DB-03: the owner favorites it → public
INSERT INTO playlist_follow (user_id, playlist_id)
SELECT user_id, (SELECT playlist_id FROM playlist WHERE name='Verify') FROM u WHERE username='ana.garcia';
SELECT visibility FROM playlist WHERE name='Verify';                 -- expect: public

-- D-10: ben favorites it, the owner unfavorites → still public
INSERT INTO playlist_follow (user_id, playlist_id)
SELECT user_id, (SELECT playlist_id FROM playlist WHERE name='Verify') FROM u WHERE username='ben.carter';
DELETE FROM playlist_follow WHERE playlist_id = (SELECT playlist_id FROM playlist WHERE name='Verify')
  AND user_id = (SELECT user_id FROM u WHERE username='ana.garcia');
SELECT visibility FROM playlist WHERE name='Verify';                 -- expect: public

-- BR-13: the last favorite goes → private
DELETE FROM playlist_follow WHERE playlist_id = (SELECT playlist_id FROM playlist WHERE name='Verify');
SELECT visibility FROM playlist WHERE name='Verify';                 -- expect: private

-- DB-03 guard: a direct change is refused
SAVEPOINT s;
UPDATE playlist SET visibility = 'public' WHERE name='Verify';       -- expect: ERROR, playlist_visibility_derived
ROLLBACK TO SAVEPOINT s;

-- DB-04: the 21st track is refused
SAVEPOINT s;
INSERT INTO playlist_track (playlist_id, track_id)
SELECT (SELECT playlist_id FROM playlist WHERE name='Verify'), track_id
FROM track ORDER BY track_id LIMIT 21;                               -- expect: ERROR, playlist_track_limit
ROLLBACK TO SAVEPOINT s;

-- DB-01: a sixth favorite is refused (uses 6 public playlists or the user's own)
-- … add 5 follows for kenji, then a sixth → expect: ERROR, playlist_follow_limit

-- DB-06 / DB-07 / DB-08: functions answer
SELECT count(*) FROM random_tracks(20);                              -- expect: 20 (or close, with gaps)
SELECT count(*) FROM random_playlists((SELECT user_id FROM u WHERE username='ana.garcia'), 12);
SELECT * FROM search_tracks('jose', 5);                               -- expect: José José tracks
SELECT DISTINCT total FROM search_tracks('daft punk', 20);           -- expect: the number of Daft Punk tracks

-- Invariant: every playlist's visibility matches its favorites
SELECT playlist_id, visibility FROM playlist p
WHERE visibility <> CASE WHEN EXISTS (SELECT 1 FROM playlist_follow f WHERE f.playlist_id = p.playlist_id)
                         THEN 'public' ELSE 'private' END;           -- expect: no rows

ROLLBACK;
```

The last query is also a good permanent check for the database project's `make verify`.

---

## Open items

| ID | Item |
|---|---|
| DBO-01 | Verify every row in [Assumptions](#assumptions-about-the-current-schema), especially DB-02 and the `track_detail` column names. |
| DBO-02 | Decide the file layout when moving to the database project: one migration per DB-xx, or folded into its schema files. Its naming and numbering apply. |
| DBO-03 | Update the seed: insert playlists without a visibility (DB-03 sets it), and decide which seed follows to keep so the example playlists stay public. Today *Road Trip* (20) has follows and stays public. |
| DBO-04 | Update `data-model.md` and the schema diagram with the new triggers, functions, view and indexes. |
| DBO-05 | `unlisted` and `share_token` stay unused until share links for private playlists exist (UI-Design.md, Future extensions). Keep the lookup value, or drop it then. |
| DBO-07 | Seed playlists with more than 20 tracks (DB-04): kept for now, and closed to new tracks. If the seed should obey the limit, trim each one to its first 20 entries and remove the exception from `make verify`. |
| DBO-06 | Duplicate tracks in a playlist (UI-Design.md Q-05): if they're forbidden, add `UNIQUE (playlist_id, track_id)` to `playlist_track`, after removing existing duplicates. |
