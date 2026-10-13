# Mixtape Bug Hunt — Submission

**Branch:** `bugfix/mixtape`

## AI Usage

_(To be completed at the end of the project.)_

---

## Codebase Map

### Files and responsibilities

| File | Responsibility |
|------|----------------|
| `app.py` | Flask **app factory**. `create_app(config)` builds the app, configures SQLAlchemy (SQLite at `instance/mixtape.db` by default, overridable via `config` — tests pass `sqlite:///:memory:`), registers the four blueprints under `/songs`, `/playlists`, `/users`, `/feed`, and runs `db.create_all()`. The module-level `db = SQLAlchemy()` is imported by every other module. Must be run with `FLASK_APP=app:create_app flask run`; running `python app.py` imports the module twice (`__main__` and `app`) and creates two `db` objects. |
| `models.py` | 7 SQLAlchemy models — `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification` — plus 3 association tables: `friendships` (User↔User, stored as two directed rows per friendship), `song_tags` (Song↔Tag), and `playlist_entries` (Playlist↔Song, with extra `position`, `added_by`, and `added_at` columns so playlist order is explicit). All primary keys are UUID strings. `Rating` has a unique constraint on `(user_id, song_id)`, so each user has at most one rating per song. Each model has a `to_dict()` used for JSON responses; `Song.to_dict()` reads tags through the `Song.tags` relationship. `User` also caches streak state directly in `listening_streak` and `last_listened_at`. |
| `routes/songs.py` | `GET /songs/search?q=` → `search_service.search_songs`; `GET /songs/<id>` → `search_service.get_song`; `POST /songs/<id>/rate` → `notification_service.rate_song`; `POST /songs/<id>/listen` → `streak_service.record_listening_event`. |
| `routes/playlists.py` | `POST /playlists/` → `playlist_service.create_playlist`; `GET /playlists/<id>` → `get_playlist`; `GET /playlists/<id>/songs` → `get_playlist_songs`; `POST /playlists/<id>/songs` → `notification_service.add_to_playlist`. |
| `routes/users.py` | `GET /users/<id>` (reads the model directly); `GET /users/<id>/streak` → `streak_service.get_streak`; `GET /users/<id>/notifications` → `notification_service.get_notifications`; `POST /users/notifications/<id>/read` → `mark_as_read`. |
| `routes/feed.py` | `GET /feed/<id>/listening-now` → `feed_service.get_friends_listening_now`; `GET /feed/<id>/activity` → `get_activity_feed`. |
| `services/streak_service.py` | Records `ListeningEvent`s and maintains the cached streak on `User`. `update_listening_streak(user, now)` compares today's calendar date to the date of `last_listened_at` and decides whether to keep, increment, or reset the streak. `get_streak` just reads the cached value. |
| `services/feed_service.py` | Friend feeds. `get_friends_listening_now` gets the user's friend IDs, queries their `ListeningEvent`s newer than a cutoff time, and keeps only the most recent event per friend. `get_activity_feed` returns the latest N friend events with no time filter. |
| `services/search_service.py` | `search_songs` runs a case-insensitive `ILIKE` match on `Song.title` or `Song.artist` with the SQLAlchemy 2.0 `select()` API; `get_song` fetches one song by ID. |
| `services/notification_service.py` | Owns notifications **and** the two "social" actions that can trigger them: `add_to_playlist` and `rate_song`. `create_notification(user_id, type, body)` is the shared helper that inserts and commits a `Notification`. `get_notifications` / `mark_as_read` read and update them. |
| `services/playlist_service.py` | Playlist CRUD: `create_playlist`, `get_playlist` (metadata only), `get_user_playlists`, and `get_playlist_songs`, which joins `Song` to `playlist_entries` and orders by `position`. |
| `seed_data.py` | Drops and recreates the DB, then inserts 5 users (nova, darius, simone, kenji, aaliya), friendships (nova↔darius/simone/kenji, darius↔simone, kenji↔aaliya), 10 tags, 13 songs with 0, 1, or 3 tags, listening events from a few minutes to ~2.5 days old, 3 playlists of 7 songs each ("Late Night Vibes", "Friday Energy", "Study Mode"), and one `song_added_to_playlist` notification. |
| `tests/` | pytest suites for streaks, search, and playlists. Each uses `create_app` with an in-memory SQLite DB, so tests never touch `mixtape.db`. |

### Data flow — a user rates a song

1. Client sends `POST /songs/<song_id>/rate` with JSON `{"user_id": ..., "score": ...}`.
2. `rate()` in `routes/songs.py` checks that both fields are present (400 if not), casts `score` to `int`, and calls `rate_song(user_id, song_id, score)` in `services/notification_service.py`.
3. `rate_song` checks that the score is 1–5, then loads the `Song` and the rating `User` with `db.session.get` and raises `ValueError` if either is missing.
4. It looks up an existing `Rating` for that `(user_id, song_id)` pair. If one exists it updates `score`; otherwise it creates a new `Rating`. This follows the model's unique constraint. Then it commits.
5. The route returns `rating.to_dict()` with status 201. Any `ValueError` from the service becomes a 400 with `{"error": ...}`.

### Data flow — a user views a playlist

`GET /playlists/<id>/songs` → `get_songs()` in `routes/playlists.py` → `get_playlist_songs(playlist_id)` in `services/playlist_service.py`. The service loads the `Playlist` (raises `ValueError` → 404 if missing), queries `Song` joined to `playlist_entries` filtered by `playlist_id` and ordered by `position` ascending, and returns a list of `song.to_dict()`. The route wraps it as `{"songs": [...], "count": n}`.

### Patterns I noticed

- **Thin routes, logic in services.** Every route only parses input, calls one service function, and formats JSON. Errors cross the boundary as `ValueError`, which routes turn into 400/404.
- **Services are grouped by side effect, not by entity.** `add_to_playlist` and `rate_song` live in `notification_service.py` rather than the playlist or songs service, because both are actions that can notify a song's sharer.
- **Notification pattern:** an action finishes its own DB write, then calls `create_notification(...)` for the song's `shared_by` user, skipping self-actions.
- **Denormalized streak.** The streak is stored on `User` and updated when a listen is recorded, not recomputed from `ListeningEvent` history.
- **All timestamps are UTC** (`datetime.now(timezone.utc)`). SQLite stores them without a timezone, so code reading them back re-attaches UTC (see `update_listening_streak`).
- Services call `db.session.commit()` themselves rather than leaving it to the route.

---

## Root Cause Analyses

### **Issue #5 — The last song in a playlist never shows up**

**How I reproduced it.** After running `seed_data.py`, I counted the rows for "Friday Energy" directly in the database (`SELECT count(*) FROM playlist_entries WHERE playlist_id=…` → **7**). Then I called `GET /playlists/<friday_energy_id>/songs`, which returned `"count": 6`. The missing song was "Harlem Renaissance", which has `position = 7`, the highest. The other two seeded playlists showed the same 7 → 6 drop. The existing tests `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` also failed (5-song playlist → 4 returned, `Track 5` missing).

**How I found the root cause.** I started at the route `get_songs()` in `routes/playlists.py`. It only calls `get_playlist_songs()` and returns `len(songs)` as the count, so the count and the list come from the same place. In `services/playlist_service.py`, I checked the query first: the join on `playlist_entries`, the filter on `playlist_id`, and `order_by(position ASC)` are all correct. That told me the database was returning all 7 rows. The last line, `return [song.to_dict() for song in songs[:-1]]`, was the moment I was sure: the data is correct until the very last step, and only then does a song disappear.

**The root cause.** `get_playlist_songs()` documents that it "returns all songs in the playlist," and its query does fetch all of them, sorted by `position` ascending. The return statement then slices the result with `songs[:-1]`, which in Python means "every element except the last one." Because the list is sorted by position, the dropped element is always the song with the highest position, which is the most recently added one. That explains the reporter's exact symptom: adding a new song makes the previously hidden song no longer last, so it reappears, and the new song is hidden instead.

**My fix.** I changed `songs[:-1]` to `songs` so the function returns the full query result. This matches its documented contract. The query and ordering were already correct, so nothing else needed to change.

**Side-effect check.**
- `pytest tests/test_playlists.py` passes 3/3, including the two tests that were failing and `test_empty_playlist_returns_empty_list`.
- Through the API, all three seeded playlists now return 7 of 7 songs in `position` order.
- A newly created empty playlist returns `{"count": 0, "songs": []}`, and an unknown playlist ID still returns 404.
- `get_playlist()` (metadata) and `add_to_playlist()` don't use this function's return value for anything except membership, so they are unaffected.

