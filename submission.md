# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude Code (claude-sonnet-4-6) throughout this project. During the codebase orientation phase, I explored both `services/` and `routes/` directories and their contents one by one, then asked the AI to read through and summarize what each function does, what queries it runs, and what it returns. Then I verified for accuracy. The summarization by AI gave me a complete picture of the architecture before I touched any issue. I also used AI to generate the architecture diagram in the codebase map section below to enable me to proceed with the codebase exploration, bug reproduction, root cause investigation, and fixes. I did not ask the AI to find bugs for me — I used the issue descriptions to narrow the search and used AI to understand code I had already located.

---

## Codebase Map

### File Inventory

**`app.py`** — Application factory. Defines the module-level `db = SQLAlchemy()` instance shared across the entire project. `create_app(config=None)` configures the database URI (env var or SQLite fallback), merges test config overrides, registers the four route blueprints at their URL prefixes, and calls `db.create_all()` at startup. Nothing else lives here.

**`models.py`** — All seven SQLAlchemy model classes and two bare association tables:

| Model / Table | Purpose |
|---|---|
| `User` | App user. Carries `listening_streak` (int) and `last_listened_at` (datetime) directly on the row. Self-referential M:M to itself via `friendships`. |
| `Song` | A song shared by a user. `shared_by` is a FK to `User`. No separate "share" model — the Song row *is* the share. Tags attached via `song_tags`. |
| `Tag` | Flat label (e.g. `"rap"`). M:M to Song via `song_tags`. |
| `ListeningEvent` | One row per play. `user_id`, `song_id`, `listened_at`. Used for feed and streak logic. |
| `Rating` | One row per (user, song) pair — unique constraint enforced. `score` stored directly; there is no separate score history. |
| `Playlist` | Named list owned by a `created_by` user. Songs attached via `playlist_entries` association table which adds `position` (sort order) and `added_by` (who added the song). |
| `Notification` | Inbox item for a user. `notification_type` (e.g. `"song_added_to_playlist"`), human-readable `body`, `read` flag. |
| `friendships` | Bare join table for the User self-join. Rows are inserted in both directions manually — the relationship is directional at the ORM level. |
| `song_tags` | Bare join table for Song ↔ Tag. No extra columns. |
| `playlist_entries` | Association table for Playlist ↔ Song with two extra non-nullable columns: `position` (int, sort order) and `added_by` (FK → User). |

**`routes/songs.py`** — Blueprint at `/songs`. Four endpoints: search songs (`GET /songs/search?q=`), get a single song, rate a song (`POST /songs/<id>/rate`), and record a listen (`POST /songs/<id>/listen`). Each handler parses HTTP input and delegates to exactly one service call.

**`routes/playlists.py`** — Blueprint at `/playlists`. Create a playlist, get playlist metadata, get playlist songs, add a song to a playlist. The add-song handler goes through `notification_service.add_to_playlist`, not `playlist_service`, because adding triggers a notification.

**`routes/users.py`** — Blueprint at `/users`. Get a user profile (direct ORM lookup, no service), get a user's streak, get a user's notifications (with optional `?unread_only=true`), mark a notification as read.

**`routes/feed.py`** — Blueprint at `/feed`. Two endpoints: `GET /feed/<user_id>/listening-now` (friends active recently) and `GET /feed/<user_id>/activity` (full recent history).

**`services/feed_service.py`** — Owns the "Friends Listening Now" and activity feed logic. `get_friends_listening_now` queries `ListeningEvent` rows for all friends within a `timedelta(hours=24)` window, then deduplicates to one event per friend (the most recent). `get_activity_feed` is similar but returns all events without deduplication, limited to 20 rows.

**`services/playlist_service.py`** — Creates playlists, retrieves playlist metadata, retrieves playlist songs (with an `ORDER BY position ASC` join query). Also defines `get_user_playlists` (unused by any route).

**`services/search_service.py`** — `search_songs` does an `outerjoin` of Song onto `song_tags` and filters by title/artist ILIKE. `get_song` is a point lookup by ID.

**`services/streak_service.py`** — `record_listening_event` creates a `ListeningEvent` row and then calls `update_listening_streak`. The streak logic compares the calendar date of the current listen against `user.last_listened_at` to decide whether to increment, leave unchanged, or reset.

**`services/notification_service.py`** — The cross-cutting service for anything that creates notifications. `add_to_playlist` adds a song to a playlist and fires a `"song_added_to_playlist"` notification to the original sharer. `rate_song` saves or updates a `Rating` row. `create_notification` is the internal helper that inserts a `Notification` row. `get_notifications` and `mark_as_read` serve the notification inbox.

**`seed_data.py`** — Drops and recreates all tables, then inserts 5 users, 10 tags, 13 songs (0 tags / 1 tag / 3+ tags), bidirectional friendship rows, listening events at varying ages, 3 playlists (7 songs each), and 1 pre-seeded notification. The seed data is specifically constructed to expose all five bugs.

**`tests/`** — Three test modules: `test_playlists.py`, `test_search.py`, `test_streaks.py`. Each uses an in-memory SQLite database via the `config` override in `create_app`.

---
