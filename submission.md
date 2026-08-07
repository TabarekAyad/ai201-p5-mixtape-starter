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

### Data Flow: Adding a Song to a Playlist (and triggering a notification)

1. Client sends `POST /playlists/<playlist_id>/songs` with JSON `{"song_id": "...", "added_by": "<user_id>"}`.
2. `routes/playlists.py` extracts both values and calls `notification_service.add_to_playlist(playlist_id, song_id, added_by_user_id)`.
3. `add_to_playlist` validates that the song, the adding user, and the playlist all exist via `db.session.get`. If any lookup fails, it raises a `ValueError` which the route catches and returns as a 400.
4. It checks whether the song is already in `playlist.songs`. If not, it appends the song and commits.
5. It then checks `song.shared_by != added_by_user_id`. If true (someone other than the original sharer is adding the song), it calls `create_notification(song.shared_by, "song_added_to_playlist", "kenji added your song 'Neon City' to the playlist 'Friday Energy'.")`.
6. `create_notification` constructs a `Notification` row, calls `db.session.add` and `db.session.commit`, and returns the instance.
7. The route returns `{"message": "Song added to playlist"}` with status 201.

The original sharer can later call `GET /users/<user_id>/notifications` — handled by `routes/users.py` → `notification_service.get_notifications` — to see the notification in their inbox.

---

### Data Flow: Recording a Listen and Updating the Streak

1. Client sends `POST /songs/<song_id>/listen` with JSON `{"user_id": "..."}`.
2. `routes/songs.py` calls `streak_service.record_listening_event(user_id, song_id)`.
3. `record_listening_event` creates a `ListeningEvent` row (not yet committed), then immediately calls `update_listening_streak(user, now)`.
4. `update_listening_streak` reads `user.last_listened_at` and computes `days_since_last` as a calendar-day difference. Based on that value it either sets the streak to 1 (first listen or gap), leaves it unchanged (already listened today), or increments it (consecutive day).
5. Back in `record_listening_event`, `db.session.commit()` persists both the new event and the updated streak in one transaction.
6. The route returns the `ListeningEvent` dict with status 201.
7. A client can verify the current streak via `GET /users/<user_id>/streak` → `routes/users.py` → `streak_service.get_streak`, which returns `user.listening_streak` directly.

---

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                           ROUTES                                 │
│          /songs     /playlists     /users     /feed              │
└───────────────────────────┬──────────────────────────────────────┘
                            │ delegates to
┌───────────────────────────▼──────────────────────────────────────┐
│                          SERVICES                                │
│    notification_service    streak_service    feed_service        │
│    playlist_service        search_service                        │
└───────────────────────────┬──────────────────────────────────────┘
                            │ queries via SQLAlchemy ORM
┌───────────────────────────▼──────────────────────────────────────┐
│                          MODELS                                  │
│                                                                  │
│  ┌─────────────────┐  M:M (friendships)  ┌─────────────────┐   │
│  │      User       │◄───────────────────►│      User       │   │
│  │─────────────────│                     └─────────────────┘   │
│  │ id              │                                            │
│  │ username        │──────────────────────────────────────┐    │
│  │ email           │ 1:M                    1:M            │    │
│  │ listening_streak│                                       │    │
│  │ last_listened_at│                                       │    │
│  └──┬──────┬───┬───┘                                       │    │
│     │1:M   │1:M│1:M                                        │    │
│     │      │   │                                           │    │
│  ┌──▼───┐  │  ┌▼──────────────────┐  M:M (song_tags)      │    │
│  │Rating│  │  │       Song        │◄────────────────►┌────┐│   │
│  │──────│  │  │───────────────────│                  │Tag ││   │
│  │score │  │  │ id, title, artist │                  └────┘│   │
│  │(1-5) │  │  │ genre, shared_by──┼──────────────────────  │   │
│  └──────┘  │  └──────────┬────────┘                        │   │
│            │             │ 1:M                              │   │
│  ┌─────────▼───┐   ┌─────▼────────────────────────────┐   │   │
│  │Listening    │   │       playlist_entries            │   │   │
│  │Event        │   │──────────────────────────────────│   │   │
│  │─────────────│   │ playlist_id (FK)  song_id (FK)   │   │   │
│  │ user_id     │   │ position          added_by (FK)  │   │   │
│  │ song_id     │   │ added_at                         │   │   │
│  │ listened_at │   └──────────────┬───────────────────┘   │   │
│  └─────────────┘                  │ M:1                    │   │
│                          ┌────────▼────────┐               │   │
│  ┌─────────────────────┐ │    Playlist     │◄──────────────┘   │
│  │    Notification     │ │─────────────────│  1:M (created_by) │
│  │─────────────────────│ │ id, name        │                   │
│  │ user_id ◄───────────┼─┤ created_by      │                   │
│  │ notification_type   │ │ is_collaborative│                   │
│  │ body, read          │ └─────────────────┘                   │
│  └─────────────────────┘                                       │
└──────────────────────────────────────────────────────────────────┘
```

---

### Patterns Noticed

**Routes are pure HTTP adapters.** Every route handler does three things only: extract inputs from `request.json` or URL params, call one service function, return a JSON response. All validation and business logic lives in services. The one exception is `GET /users/<user_id>`, which does a direct `db.session.get` in the route — the only place in the codebase where a route touches the ORM directly.

**`notification_service` is the cross-cutting service.** It is the only service module imported by more than one route file (`routes/playlists.py` and `routes/songs.py` both import from it). This is because both playlist additions and song ratings are actions that create notifications, so the notification side-effects are bundled into the same service functions that perform the underlying writes.

**Association tables with extra columns are handled inconsistently.** `playlist_entries` has `position` and `added_by` columns that the ORM `relationship` cannot populate via a simple `.append()`. The seed data bypasses the ORM and inserts rows directly with `db.session.execute(playlist_entries.insert(), {...})`. The service layer does not follow this pattern, which is the root cause of bugs involving playlist entries.

**UUIDs everywhere.** Every model generates its own UUID primary key via `default=generate_uuid`. There are no auto-increment integers in the schema. This means IDs must be provided or retrieved — you cannot guess them.

**All times are UTC.** `datetime.utcnow` is used throughout for defaults and comparisons. The streak and feed logic both depend on UTC-aware datetime arithmetic. Any confusion between naive and aware datetimes (or between UTC calendar days and local calendar days) surfaces as a bug.

---

## Bug Fixes

---

### Issue #1 — My listening streak keeps resetting

**1. How I reproduced it**

The seed data sets kenji's `last_listened_at` to 3 hours ago (today), which does not trigger the bug because same-day listens are a no-op in the streak logic. The bug only fires on a Sunday, and today is Thursday 2026-08-06, so a live HTTP call would never reach the Sunday branch. To match the reported condition — listening on Saturday then recording on Sunday — I froze the clock at Sunday 2026-08-02 using `unittest.mock.patch` and drove the service directly.

`tmp_repro.py` does three things: resets kenji's state to streak=12 / last_listened_at=2026-08-01 (Saturday) via the ORM, patches `datetime.now` inside `services.streak_service` to return a fixed Sunday datetime (2026-08-02 12:00 UTC), then calls `record_listening_event` through the real service code:

```python
# tmp_repro.py
from unittest.mock import patch, MagicMock
from datetime import datetime, timezone

SUNDAY = datetime(2026, 8, 2, 12, 0, 0, tzinfo=timezone.utc)

from app import create_app
from models import User
from app import db

app = create_app()

with app.app_context():
    kenji = db.session.query(User).filter_by(username='kenji').first()
    kenji.listening_streak = 12
    kenji.last_listened_at = datetime(2026, 8, 1, 12, 0, 0, tzinfo=timezone.utc)
    db.session.commit()
    print(f"Starting state: streak={kenji.listening_streak}, last_listened_at={kenji.last_listened_at.date()}")

    mock_dt = MagicMock(wraps=datetime)
    mock_dt.now.return_value = SUNDAY

    with patch('services.streak_service.datetime', mock_dt):
        from services import streak_service
        streak_service.record_listening_event(kenji.id, '290f46f8-5414-4de8-a588-4568a10cab7f')

    db.session.refresh(kenji)
    print(f"After Sunday listen: streak={kenji.listening_streak}  (expected 13)")
```

```powershell
python tmp_repro.py
```

Output:
```
Starting state: streak=12, last_listened_at=2026-08-01
After Sunday listen: streak=1  (expected 13)
```

Bug confirmed: with the clock frozen at Sunday 2026-08-02, recording a listen after a Saturday sets the streak to 1 instead of 13.

**2. How I found the root cause**

I opened [services/streak_service.py](services/streak_service.py) because the architecture map names it as the only file that owns streak logic. I read `update_listening_streak` top to bottom. The function computes `days_since_last` and branches on it. The branch that should increment the streak caught my eye immediately:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

`datetime.date.weekday()` returns 0–6 (Monday=0, Sunday=6). The extra guard `!= 6` means "and today is not Sunday." That was the moment of confidence — on a Saturday → Sunday listen, `days_since_last == 1` is True but `today.weekday() != 6` is False, so the condition fails and falls through to `else`, which resets the streak to 1. The weekday check has no place in streak logic at all.

**3. The root cause**

`datetime.date.weekday()` returns 6 for Sunday. The `elif` at [services/streak_service.py:73](services/streak_service.py#L73) was:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
```

The streak rule is purely about consecutive calendar days — the day of the week is irrelevant. The extra guard `and today.weekday() != 6` made Sunday the one day of the week where a consecutive-day listen was silently blocked. When `days_since_last == 1` and today is Sunday (weekday == 6), the condition evaluated to `False`, bypassed the increment, and fell through to `else: user.listening_streak = 1`, resetting the streak to 1 instead of incrementing it.

**4. Fix and side-effect check**

Removed `and today.weekday() != 6` from the condition:

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

Verified with `tmp_repro.py` — clock frozen at Sunday 2026-08-02, last listened Saturday 2026-08-01, streak started at 12. After the fix, streak correctly incremented to 13. The `days_since_last == 0` (same-day no-op) and `else` (gap reset) branches are structurally unchanged. No other code in the project calls `weekday()` on a date, so there are no side effects outside this function.

---

### Issue #3 — The same song keeps showing up twice in search

**1. How I reproduced it**

No database setup required. I searched for "Anthem":

```
GET http://127.0.0.1:5000/songs/search?q=Anthem
```

```powershell
(Invoke-WebRequest "http://127.0.0.1:5000/songs/search?q=Anthem").Content | python -m json.tool
```

The endpoint returned **1 result** for Crown Heights Anthem — not 3 as the issue report described. Investigation revealed why: the bug is real at the SQL level but masked at the Python level by SQLAlchemy 2.0.

`search_service.py` uses an `outerjoin` on `song_tags` without `.distinct()`. The underlying SQL produces 3 rows for "Crown Heights Anthem" (one per tag: `rap`, `hip-hop`, `boom bap`). However, SQLAlchemy 2.0's legacy `Query` API (`db.session.query(Song)`) automatically deduplicates ORM entity results by primary key before returning — all 3 rows share the same `song.id`, so `.all()` hands back a list with 1 `Song` object instead of 3.

The test comment at `tests/test_search.py` says `# Should be 1, bug causes it to be 3` — that comment was written for SQLAlchemy 1.x behavior where `.all()` would return 3 identical references. With SQLAlchemy 2.0 the ORM masks the bug at the Python level, so the test passes vacuously. The underlying SQL is still wrong and would produce duplicates in any context that bypasses the ORM deduplication (raw SQL, different SQLAlchemy versions, or certain query patterns).

**Expected:** each matching song appears exactly once, and the SQL query produces exactly one row per song.  
**Actual:** the API response is correct by accident — SQLAlchemy 2.0 silently collapses the duplicate rows the bad `outerjoin` produces.

**2. How I found the root cause**

*(To be completed in Milestone 3.)*

**3. The root cause**

*(To be completed in Milestone 3.)*

**4. Fix and side-effect check**

*(To be completed in Milestone 3.)*

---

### Issue #5 — The last song in a playlist never shows up

**1. How I reproduced it**

No manual database setup was needed — the seed data is sufficient. The "Friday Energy" playlist is seeded with 7 songs (`all_songs[3:10]` in `seed_data.py`).

Retrieved the playlist and user IDs using Flask shell:

```python
from models import User, Playlist
for u in User.query.all():
    print(u.id, u.username)
for p in Playlist.query.all():
    print(p.id, p.name)
```

Then fetched the playlist songs:

```
GET http://127.0.0.1:5000/playlists/437c6ebc-ea8f-4037-a1cb-4727e918c096/songs
```

Response:

```json
{"count": 6, "songs": [...]}
```

**Expected:** `"count": 7` — the seed inserted 7 songs into Friday Energy.  
**Actual:** `"count": 6` — the most recently added song (highest `position`) is always missing.

To confirm the "sliding" behavior, I tried `POST /playlists/<playlist_id>/songs` to add a new song, but it returned a 500 error. The cause: `notification_service.add_to_playlist` uses `playlist.songs.append(song)` through the ORM relationship, but the `playlist_entries` association table has two `NOT NULL` columns without defaults (`position` and `added_by`) that the ORM cannot populate via `.append()`. This is the same reason the seed data bypasses the ORM entirely and inserts rows with `db.session.execute(playlist_entries.insert().values(...))`.

To work around this, I added "After Hours" directly to the database using the same pattern as the seed:

```python
import sqlite3
from datetime import datetime, timezone

PLAYLIST_ID = "437c6ebc-ea8f-4037-a1cb-4727e918c096"  # Friday Energy
SONG_ID     = "b5719902-c390-47dd-826f-4b551f53b1ff"  # After Hours
ADDED_BY    = "fe7b8832-7b2d-4766-a9f5-ca592c0d2ac9"  # darius

conn = sqlite3.connect("instance/mixtape.db")
max_pos = conn.execute(
    "SELECT MAX(position) FROM playlist_entries WHERE playlist_id = ?", (PLAYLIST_ID,)
).fetchone()[0]
conn.execute(
    "INSERT INTO playlist_entries (playlist_id, song_id, position, added_by, added_at) VALUES (?, ?, ?, ?, ?)",
    (PLAYLIST_ID, SONG_ID, max_pos + 1, ADDED_BY, datetime.now(timezone.utc).isoformat())
)
conn.commit()
```

The database now had 8 songs in Friday Energy. Re-fetching the endpoint returned `"count": 7` — "Harlem Renaissance" (previously missing at pos 7) now appeared, and "After Hours" (pos 8) became the new missing song. That confirmed the sliding behavior exactly as darius described.

**2. How I found the root cause**

*(To be completed in Milestone 3.)*

**3. The root cause**

*(To be completed in Milestone 3.)*

**4. Fix and side-effect check**

*(To be completed in Milestone 3.)*

---

*Branch: `bugfix/mixtape`*
