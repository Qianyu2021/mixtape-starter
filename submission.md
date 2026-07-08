# Mixtape — Codebase Map

Mixtape is a Flask + SQLAlchemy REST API for a social music-sharing app. Users share songs, add them to collaborative playlists, rate them, listen to them (building daily streaks), and see what their friends are listening to. There is no frontend in this repo — it's a JSON API.

## Main files and what each does

### `app.py` — application factory + DB handle
Defines the shared `db = SQLAlchemy()` instance and a `create_app()` factory. The factory reads config (`DATABASE_URL`, defaulting to a local SQLite file `mixtape.db`), registers the four route blueprints under URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()`. Every other module imports `db` from here, so this is the root of the dependency graph.

### `models.py` — all database entities
Defines 7 SQLAlchemy models plus 3 association tables:

- **`User`** — id (UUID string), username, email, `listening_streak`, `last_listened_at`. Owns relationships to shared songs, ratings, listening events, notifications, and playlists. Friendships are a **self-referential many-to-many** via the `friendships` table (symmetric friend list, `lazy="dynamic"`).
- **`Song`** — title, artist, album, genre, plus `shared_by` (FK to the User who shared it), `shared_at`, and `share_note`. The `shared_by` field is what makes the notification system work — it's the "owner" of the song.
- **`Tag`** — a name; linked to songs many-to-many via `song_tags`.
- **`ListeningEvent`** — one row per listen (user_id, song_id, `listened_at`). This is the **event log that powers the feed and streaks** — there's no denormalized "feed" table; feeds are queries over this log.
- **`Rating`** — user_id, song_id, score (1–5), with a `UniqueConstraint(user_id, song_id)` so a user has at most one rating per song (re-rating updates in place).
- **`Playlist`** — name, `created_by`, `is_collaborative` flag. Songs attach through the `playlist_entries` association table.
- **`Notification`** — user_id (recipient), `notification_type` string, `body` text, `read` boolean. Generic: type + free-text body rather than a per-event schema.

**The three association tables carry real data, not just FK pairs:**
- `friendships` — (user_id, friend_id).
- `song_tags` — (song_id, tag_id).
- `playlist_entries` — (playlist_id, song_id) **plus `position` (Integer), `added_by`, and `added_at`**. Playlist membership records *who* added a song, *when*, and its *explicit ordering position* — songs in a playlist have a defined order, not just insertion order.

### `routes/` — HTTP layer (4 blueprints)
Thin controllers. Each parses the request (JSON body or query params), calls exactly one service function, and formats the JSON response / maps `ValueError` to a 4xx status.
- **`songs.py`** — `GET /songs/search`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`.
- **`playlists.py`** — `POST /playlists/`, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`.
- **`users.py`** — `GET /users/<id>`, `GET /users/<id>/streak`, `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read`.
- **`feed.py`** — `GET /feed/<user_id>/listening-now`, `GET /feed/<user_id>/activity`.

### `services/` — business logic (5 modules)
All logic lives here; routes never touch the DB directly (except `users.py` doing a simple `get(User, ...)` existence check).
- **`notification_service.py`** — `create_notification`, `add_to_playlist` (the write path that also fires a notification), `rate_song`, `get_notifications`, `mark_as_read`.
- **`feed_service.py`** — `get_friends_listening_now` (friends' listens in the last 24h, deduped to one song per friend) and `get_activity_feed` (most recent N friend listens, no recency filter).
- **`playlist_service.py`** — `create_playlist`, `get_playlist_songs`, `get_playlist`, `get_user_playlists`.
- **`search_service.py`** — `search_songs` (case-insensitive title/artist match) and `get_song`.
- **`streak_service.py`** — `record_listening_event` (logs a listen + updates streak) and `update_listening_streak` (consecutive-day streak logic), `get_streak`.

### Supporting files
- **`seed_data.py`** — populates demo data.
- **`tests/`** — `test_playlists.py`, `test_search.py`, `test_streaks.py`.

## Data flow — adding a song to a playlist triggers a notification

This is the app's clearest cross-model flow (the equivalent of the "sharing triggers a notification" example — note there is no standalone *share* endpoint; songs get a `shared_by` owner at creation, and playlist-adds are what notify that owner):

1. **Request** — `POST /playlists/<playlist_id>/songs` with JSON `{song_id, added_by}` hits `add_song()` in [routes/playlists.py:44](routes/playlists.py#L44). The route validates both fields are present, otherwise returns 400.
2. **Delegate to service** — the route calls `add_to_playlist(playlist_id, song_id, added_by)` in [services/notification_service.py:35](services/notification_service.py#L35).
3. **Load + validate entities** — the service fetches the `Song`, the adding `User`, and the `Playlist`, raising `ValueError` (→ 400) if any is missing.
4. **Mutate the playlist** — if the song isn't already in `playlist.songs`, it's appended and committed (this writes a `playlist_entries` row).
5. **Conditional notification** — `if song.shared_by != added_by_user_id:` the service calls `create_notification(user_id=song.shared_by, type="song_added_to_playlist", body="<adder> added your song '<title>' to the playlist '<name>'.")`. A user adding their **own** song gets no notification.
6. **Persist notification** — `create_notification` ([notification_service.py:13](services/notification_service.py#L13)) builds a `Notification` row and commits it.
7. **Read side** — later, the sharer calls `GET /users/<id>/notifications` → `get_notifications()`, which queries `Notification` filtered by recipient, newest first.

The rating flow (`POST /songs/<id>/rate` → `rate_song`) is structured identically but currently does **not** create a notification, even though `create_notification`'s docstring lists `song_rated` as an expected type — a notification call appears to be missing there.

## Patterns in how the app is organized

- **Strict route → service → model layering.** Routes parse input and format responses; services own all business logic and DB access. Blueprints are registered with URL prefixes in the app factory, so URL structure is centralized in `app.py`.
- **`ValueError` as the error protocol.** Services raise `ValueError` for "not found" / bad input; every route wraps its call in `try/except ValueError` and maps it to a 400 or 404. There are no custom exception types.
- **Application-factory + shared `db`.** `db` is created in `app.py` and imported everywhere, with `create_app(config=None)` allowing test config injection (`SQLALCHEMY_DATABASE_URI` override for an in-memory test DB).
- **UUID string primary keys everywhere**, generated in Python via `generate_uuid()` defaults rather than DB auto-increment.
- **`to_dict()` on every model** as the serialization boundary — services return `to_dict()` output (or model instances the route then serializes), so routes just `jsonify`. Note `User.to_dict()` deliberately omits `email` and `created_at`.
- **Event-sourced feed/streak, generic notifications.** The feed and streak are computed from the `ListeningEvent` log rather than stored, while notifications are a single generic table keyed by a `notification_type` string + free-text `body` — new notification kinds need no schema change.
- **Association tables carry attributes** (`playlist_entries.position/added_by/added_at`), showing many-to-many relationships are treated as first-class records, not just links.
- **Lazy cross-service imports to avoid cycles.** `add_to_playlist` imports `Playlist` and `playlist_service` *inside* the function, a deliberate workaround for the `notification_service` ↔ `playlist_service` import cycle.

---

# Debugging

## Issue #1 — "My listening streak keeps resetting"

Reported symptom: *"On Saturday night my streak was at 12. Sunday morning I played a song like always, checked my profile, and my streak said 1."*

### How I reproduced it

Before touching any code I recreated the exact reported sequence against `update_listening_streak(user, now)`:

1. Start with a user whose `listening_streak = 12` and `last_listened_at` = **Saturday** 2026-07-04 20:00 UTC.
2. Call the streak update with `now` = **Sunday** 2026-07-05 08:00 UTC (the "Sunday morning listen").
3. Expected result: streak → 13 (consecutive day). Actual result: streak → **1**.

That reproduced the reset. I then varied the day-of-week: the reset only happened when the *new* listen landed on a **Sunday**. A Saturday→Sunday, Monday→(any weekday), or Sunday→Monday sequence behaved fine — only "today is Sunday" triggered the bad reset. That day-of-week dependency was the key signal that the bug was calendar-based, not a general off-by-one in the day math.

### How I found the root cause

- Started from the README issue table, which pointed me at `services/streak_service.py`.
- Read `update_listening_streak` — the only place the streak is incremented or reset. The consecutive-day math itself (`days_since_last = (today - last_date).days`) looked correct and matched the docstring.
- The moment of confidence was line 73: the `elif days_since_last == 1 and today.weekday() != 6:` branch. The docstring lists four streak rules and **none of them mention weekends or Sundays** — yet there was an extra `and today.weekday() != 6` clause gating the increment. `datetime.weekday()` returns `6` for Sunday, so this clause is false exactly when today is Sunday. That single stray condition explained the day-of-week pattern I saw in reproduction — not just a suspicious area, but the precise line that forces the `else` (reset) branch on Sundays.

### The root cause

The increment branch required both `days_since_last == 1` **and** `today.weekday() != 6`. When a user listened on consecutive days but the second listen fell on a **Sunday** (`weekday() == 6`), the compound condition evaluated to false. Execution then fell through to the `else` branch, which sets `listening_streak = 1`. So any streak crossing into a Sunday was wiped out and restarted at 1, regardless of how long it had been. The `!= 6` check had no basis in the documented rules — it was an erroneous condition that treated Sundays as a streak break.

### The fix and side-effect check

Changed [services/streak_service.py:73](services/streak_service.py#L73):

```diff
- elif days_since_last == 1 and today.weekday() != 6:
+ elif days_since_last == 1:
```

Removing the `and today.weekday() != 6` clause makes the increment depend only on whether the previous listen was exactly one calendar day ago — which is what the docstring's rules describe. This fixes the root cause because Sundays are no longer singled out for a reset.

To confirm nothing else broke, I exercised the function directly across the neighboring cases and printed the results:

```
Sat(12) -> Sun listen  => streak = 13 (expected 13)   # the reported bug — now fixed
gap of 4 days          => streak = 1  (expected 1)    # a skipped day still resets
same-day 2nd listen    => streak = 7  (expected 7)    # multiple listens same day = no change
first-ever listen      => streak = 1  (expected 1)    # never-listened user starts at 1
```

All four of the documented streak rules still hold, and the reset-on-gap and same-day-no-op behaviors are unaffected — the only behavior that changed is that consecutive-day listens landing on a Sunday now correctly increment.

## Issue #2 — "Friends Listening Now shows people from yesterday"

Reported symptom: the "Friends Listening Now" feed lists friends who listened hours ago — even people who last listened *yesterday* — instead of only those listening right now.

### How I reproduced it

Before changing code I set up three friends with listens at different ages and called `get_friends_listening_now(user_id)`:

1. Friend A listened **10 minutes ago** → should appear ("listening now").
2. Friend B listened **2 hours ago** (earlier today) → should NOT appear.
3. Friend C listened **18 hours ago** (yesterday) → should NOT appear.

With the original code all three came back:

```
BEFORE (24h)   -> ['recent_10min', 'older_2h', 'yesterday_18h']
```

Friends B and C are exactly the "people from yesterday" the report complained about, confirming the bug.

### How I found the root cause

- The README issue table pointed me at `services/feed_service.py`.
- I read `get_friends_listening_now` and saw the recency filter `ListeningEvent.listened_at >= cutoff`, where `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`. The SQL filter and dedup logic were correct, so the only thing that could widen the window was the threshold constant itself.
- The deciding moment was the module-level constant `RECENT_THRESHOLD = timedelta(hours=24)` combined with the intent baked into `seed_data.py`. The seed file explicitly documents the expected window: recent events are created "within the past 30 minutes — should appear in 'listening now'" ([seed_data.py:111](seed_data.py#L111)) while older events are "1–14 days ago — should NOT appear in 'listening now' after fix" ([seed_data.py:121](seed_data.py#L121)). That mismatch — a 24-hour threshold versus a documented 30-minute window — pinpointed the exact defect, not just a suspicious area.

### The root cause

`RECENT_THRESHOLD` was set to **24 hours**. "Friends Listening Now" is supposed to show only friends listening *right now* (within the last 30 minutes), but a 24-hour cutoff means any friend who listened at any point in the previous day still satisfies `listened_at >= now - 24h`. So a friend who played a song yesterday evening keeps appearing in the "now" feed all the way until 24 hours later. The window was simply ~48× too large for what the feature promises.

### The fix and side-effect check

Changed [services/feed_service.py:13](services/feed_service.py#L13):

```diff
- RECENT_THRESHOLD = timedelta(hours=24)
+ RECENT_THRESHOLD = timedelta(minutes=30)
```

This matches the 30-minute window the seed data documents, so events older than 30 minutes (earlier today or yesterday) are excluded while genuinely-current listens still show. Re-running the reproduction:

```
AFTER (30min)  -> ['recent_10min']            # only the friend listening now
activity_feed  -> ['recent_10min', 'earlier_today_2h', 'yesterday_18h']   # unchanged
dedup: entries for recent friend = 1          # one row per friend, most recent
```

Side-effect checks:
- **`get_activity_feed`** — intentionally *not* recency-filtered — still returns older events (it only orders by recency and limits count), so the general activity history is unaffected by the threshold change.
- **Dedup / ordering** in `get_friends_listening_now` still returns exactly one entry per friend (their most recent listen), so the fix narrows the time window without disturbing how results are grouped or sorted.
- **Empty cases** (no friends / no recent listens) still return `[]`.

## Issue #3 — "The same song keeps showing up twice in search"

Reported symptom: searching for a song returns the same song multiple times in the results list.

### How I reproduced it

Before touching code I created a song with three tags and searched for it. The `search_songs` query joins the `song_tags` association table, so at the SQL level the row set fans out to one row per tag. I confirmed this directly:

```
raw SQL rows for a 3-tag song matching the query: 3   # 'Crown Heights Anthem' x3
```

The bundled test `tests/test_search.py::test_search_no_duplicates_multi_tag_song` documents the same expectation ("Should be 1, bug causes it to be 3"). One nuance worth recording: on the installed SQLAlchemy (2.0.51), the *legacy* `Query.all()` path happens to de-duplicate whole-entity results by identity, which masks the duplicate when selecting full `Song` objects. But the fan-out is real and surfaces the moment uniquing doesn't apply — e.g. selecting a column:

```
title rows via join (buggy): ['Crown Heights Anthem', 'Crown Heights Anthem', 'Crown Heights Anthem']
```

…and it would surface directly to users on any SQLAlchemy version or query style (2.0 `select().scalars()`, `.count()`, older releases) that doesn't silently unique the rows. So the query was latently broken regardless of the accidental masking.

### How I found the root cause

- The README issue table pointed me at `services/search_service.py`.
- Reading `search_songs`, the filter only matches on `Song.title` / `Song.artist` — yet the query also did `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`. Nothing in the query references the joined `song_tags` rows for filtering or selection.
- The confirming moment was checking `Song.to_dict()` in `models.py`: tags are serialized via `self.tags`, which is a `db.relationship("Tag", secondary=song_tags, lazy="subquery")` ([models.py:90](models.py#L90)). Tags are loaded by that relationship completely independently of the search query. That proved the `outerjoin` in `search_songs` contributes nothing to the output — its *only* effect is to multiply result rows by each song's tag count. That's the precise cause, not just a suspicious line.

### The root cause

`search_songs` performed an `OUTER JOIN` against the `song_tags` association table but never used it — no tag filter, no tag columns selected. A relational join emits one row per matching pair, so a song with N tags produced N identical rows. The intent was presumably to make tags available, but tags already come from the `Song.tags` relationship inside `to_dict()`, making the join both redundant and the direct source of the duplicates.

### The fix and side-effect check

Changed [services/search_service.py](services/search_service.py) — removed the unused join (and the now-unused `Tag`, `song_tags` imports):

```diff
- from models import Song, Tag, song_tags
+ from models import Song

  results = (
      db.session.query(Song)
-     .outerjoin(song_tags, Song.id == song_tags.c.song_id)
      .filter(
          db.or_(
              Song.title.ilike(f"%{query}%"),
              Song.artist.ilike(f"%{query}%"),
          )
      )
      .all()
  )
```

Without the join the query returns exactly one row per matching song, so no version- or query-style-dependent uniquing is required to get correct results.

Side-effect checks:
- **Tags still present** — a search result for the 3-tag song still includes `['rap', 'hip-hop', 'boom bap']`, confirming `to_dict()`'s relationship-based tag loading is unaffected by removing the join.
- **Search test suite** — all 5 tests in `tests/test_search.py` pass (matching, empty-result, and the no-duplicate cases for zero/one/three tags).
- **Duplicate gone at the row level** — the previously-duplicating scenario now returns a single row, so the fix holds even where entity uniquing wouldn't have saved it.
- The two failures in `tests/test_playlists.py` are unrelated to this change (they stem from the still-open Issue #5 in `playlist_service.py`, which this fix does not touch).
