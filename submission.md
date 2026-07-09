# Mixtape Submission

## Milestone 1: Codebase Map

### Main files and their roles

- **`app.py`**: Flask application factory (`create_app`). Initializes `SQLAlchemy` (`db`), loads config (DB URI, secret key), registers the four blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and calls `db.create_all()`. This is also where the `db` object that every model and service imports lives. That's why the app must be started with `flask run` (via `FLASK_APP=app:create_app`) rather than `python app.py`: importing `app.py` directly can trigger a circular/double-import of `db`, since `models.py` imports `db` from `app`.

- **`models.py`**: Defines all SQLAlchemy models and association tables.
  - `User`: has `listening_streak` and `last_listened_at` fields used by the streak feature, a symmetric many-to-many `friends` relationship (via the `friendships` table), and backrefs to songs shared, ratings given, listening events, notifications, and playlists created.
  - `Song`: a shared song, owned by the user who shared it (`shared_by`), with many-to-many `tags` (via `song_tags`).
  - `ListeningEvent`: one row per "user listened to song," timestamped `listened_at`. This is the source of truth for streaks and the activity/listening-now feeds.
  - `Rating`: a user's 1 to 5 score for a song, unique per `(user_id, song_id)`.
  - `Playlist`: has a `songs` many-to-many relationship via the `playlist_entries` table, which is the join table that also carries `position` (explicit ordering), `added_by`, and `added_at`. Songs are not just appended; their order in a playlist is determined by `position`, not insertion order.
  - `Notification`: a per-user message with a `notification_type` (e.g. `song_added_to_playlist`), `body`, and `read` flag.

- **`routes/`**: Thin HTTP layer. Every route parses the request, calls exactly one service function, and formats the JSON response. No business logic lives here.
  - `songs.py`: `/songs/search`, `/songs/<id>`, `/songs/<id>/rate`, `/songs/<id>/listen`
  - `playlists.py`: `/playlists/` (create), `/playlists/<id>`, `/playlists/<id>/songs` (GET/POST)
  - `users.py`: `/users/<id>`, `/users/<id>/streak`, `/users/<id>/notifications`, `/users/notifications/<id>/read`
  - `feed.py`: `/feed/<id>/listening-now`, `/feed/<id>/activity`

- **`services/`**: All business logic. Each file owns one concern and is where the real behavior (and the reported bugs) lives.
  - `streak_service.py`: `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares `now.date()` to `user.last_listened_at.date()` to decide whether to increment (consecutive day), no-op (same day), or reset (gap) the streak.
  - `feed_service.py`: `get_friends_listening_now()` finds the user's friends, pulls their `ListeningEvent`s within a rolling `RECENT_THRESHOLD` (24 hours) of "now," dedupes to the most recent event per friend, and returns them. `get_activity_feed()` is the non-recency-filtered version, returning the most recent N events regardless of age.
  - `search_service.py`: `search_songs()` matches title/artist case-insensitively and joins against the `song_tags` table to pull in tags. `get_song()` fetches a single song by id.
  - `notification_service.py`: `create_notification()` is the generic notification writer. `add_to_playlist()` adds a song to a playlist's `songs` collection and then calls `create_notification()` to tell the original sharer (unless they added it themselves). `rate_song()` saves or updates a `Rating` but does not call `create_notification()` anywhere. `get_notifications()` and `mark_as_read()` are read/update helpers for the notification list.
  - `playlist_service.py`: `create_playlist()`, `get_playlist()` (metadata only), `get_user_playlists()`, and `get_playlist_songs()`, which joins `Song` to `playlist_entries` filtered by `playlist_id` and ordered by `position` ascending.

- **`seed_data.py`**: Populates the DB with sample users, friendships, songs, playlists, ratings, and listening events for local testing.

- **`tests/`**: `test_streaks.py`, `test_search.py`, `test_playlists.py` cover (partially) the streak, search, and playlist behaviors.

### Data flow: rating a song to notification (traced end-to-end)

1. Client sends `POST /songs/<song_id>/rate` with `{ "user_id": ..., "score": ... }`.
2. `routes/songs.py::rate()` parses the body and calls `notification_service.rate_song(user_id, song_id, score)`.
3. `rate_song()` validates the score range, looks up the `Song` and `User`, then either updates an existing `Rating` row (unique per user/song) or inserts a new one, and commits.
4. The route returns the serialized `Rating` (HTTP 201).

Contrast with the parallel flow for adding a song to a playlist:

1. Client sends `POST /playlists/<playlist_id>/songs` with `{ "song_id": ..., "added_by": ... }`.
2. `routes/playlists.py::add_song()` calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. `add_to_playlist()` appends the song to `playlist.songs` (if not already present), commits, and, if the adder isn't the song's original sharer, calls `create_notification()` to notify `song.shared_by`.

**Pattern noticed:** every write-side action that's supposed to notify a song's original sharer should call `notification_service.create_notification()` after the write commits. `add_to_playlist()` follows this pattern; `rate_song()` does not call `create_notification()` at all, it only writes the `Rating` row. Any "notify the sharer" feature has to be implemented explicitly per action, it isn't automatic.

### Five open issues (read, not yet triaged for which 3 to fix)

| # | Title | Affected service |
|---|-------|-------------------|
| 1 | Listening streak resets even with consecutive days (specifically on Sundays) | `streak_service.py` |
| 2 | Friends Listening Now shows people from yesterday evening the next morning | `feed_service.py` |
| 3 | Same song appears multiple times in search results | `search_service.py` |
| 4 | Rating a song doesn't notify the sharer, unlike adding to a playlist | `notification_service.py` |
| 5 | The most recently added song in a playlist never shows up in the songs list | `playlist_service.py` |

### AI-assisted orientation

Used Claude Code to read each service file, summarize its responsibility, and trace the rate-to-notification and add-to-playlist-to-notification call chains side by side to compare the two patterns. This orientation pass was used only to build a mental model of the codebase structure and data flow. No bug fixes were made or attempted during this milestone.
