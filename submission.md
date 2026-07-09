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

## Milestone 2: Reproducing the Bugs

I picked three bugs to fix: #1 (streak), #2 (listening now), and #4 (rating notification). I originally planned to fix #3 (duplicate search results) instead of #2, but I couldn't actually reproduce it, more on that below, so I swapped it out.

### Issue #1: streak resets on Sunday

**How I reproduced it:** today's real date isn't a Sunday, so I couldn't trigger this through the live app clock. Instead I called `update_listening_streak()` directly with two crafted, consecutive dates: a Saturday evening listen, then a Sunday morning listen for the same user. Both listens are one calendar day apart, so the streak should have gone from 1 to 2. Instead it stayed at 1 after the Sunday listen.

This lines up with the code in `streak_service.py`: the increment branch only fires when `days_since_last == 1 and today.weekday() != 6`. On a Sunday, `today.weekday()` is `6`, so that condition is false even though the user listened on consecutive days, and the streak falls through to the reset branch instead.

### Issue #2: "Friends Listening Now" shows stale entries

**How I reproduced it:** I wanted to reproduce the exact scenario from the report (a friend listens late at night, and is still shown as "listening now" the next morning) without waiting for real time to pass. I set up a friend who listened at 11pm on a given day, then mocked the clock to check the feed at 9am the next day, only 10 hours later but a different calendar day. The friend still showed up in the "listening now" results.

The root cause is in `feed_service.py`: `get_friends_listening_now()` filters listening events using a rolling 24-hour window (`RECENT_THRESHOLD`), not "did this happen today" by calendar date. Anything within the last 24 hours counts as recent, so a late-night listen keeps showing up well into the next morning until the rolling window finally passes it.

### Issue #4: rating a song doesn't notify the sharer

**How I reproduced it:** this one didn't need any mocking, it reproduces directly through the running app. I had one user (simone) rate a song shared by another user (darius) via `POST /songs/<song_id>/rate`. The rating saved correctly (score 5, returned with a 201). I then checked darius's notifications with `GET /users/<darius_id>/notifications` and got back an empty list.

Looking at `notification_service.py`, `add_to_playlist()` calls `create_notification()` after adding a song to a playlist, but `rate_song()` never does. It only writes the `Rating` row and returns. There's no missing condition or typo, the call to `create_notification()` for ratings simply doesn't exist yet.

### A bug that didn't reproduce: Issue #3 (duplicate search results)

Before settling on #2, I tried to reproduce Issue #3 first, since the code in `search_service.py` looked like an obvious candidate: `search_songs()` does an `outerjoin` against `song_tags` without a `.distinct()`, and the seed data has a song ("Crown Heights Anthem") with three tags, exactly the setup the bug report describes.

I tried it three ways: hitting `GET /songs/search?q=Anthem` on the live app, calling `search_songs()` directly in a script, and running the existing test `test_search_no_duplicates_multi_tag_song`. All three returned the song exactly once. No duplicates.

I dug a bit further and confirmed this isn't specific to this codebase's setup: I reproduced the same join in a minimal, standalone SQLAlchemy script with no Flask involved, and it also returned one row, even though the raw SQL underneath genuinely returns three. The installed SQLAlchemy version (2.0.51) deduplicates full-entity `Query.all()` results by primary key, even when a join fans out the underlying row count. So the missing `.distinct()` is still a latent issue in the code (it would bite you if you ever selected extra columns from the join, or dropped down to raw SQL), but it doesn't actually produce duplicate results in this app as it's set up right now.

Since I couldn't reproduce the reported behavior after a genuine attempt, and confirmed the seed data was correct, I swapped in Issue #2 in its place per the milestone's guidance.

## Milestone 3: Root Cause Analysis and Fixes

### Issue #1: My listening streak keeps resetting

**How I reproduced it:** covered in Milestone 2 above. Called `update_listening_streak()` directly with a Saturday listen followed by a Sunday listen, one calendar day apart. The streak should go up by 1 but stayed flat instead.

**How I found the root cause:** the docstring for `update_listening_streak()` lays out the rule plainly: same day means no change, one day apart means increment, more than one day means reset. There's no mention of any day-of-week exception. Reading the actual `if/elif/else` block against that docstring, the `elif` branch had an extra condition tacked on, `and today.weekday() != 6`, that isn't described anywhere in the rules above it. That's the kind of mismatch between stated intent and code that's worth chasing: the increment branch should only ever check `days_since_last == 1`.

**The root cause:** in `services/streak_service.py`, the increment condition reads `elif days_since_last == 1 and today.weekday() != 6:`. Python's `datetime.weekday()` returns `6` for Sunday. So on any Sunday, even if the user listened on both Saturday and Sunday (a genuine consecutive day), that `and` clause evaluates to false, the `elif` is skipped, and execution falls through to the `else` branch, which resets the streak to 1 instead of incrementing it.

**My fix and side-effect check:** removed the `and today.weekday() != 6` clause, so the branch now reads `elif days_since_last == 1:`. This is a one-line change scoped only to the increment condition, it doesn't touch the same-day no-op branch or the reset branch. I reran the full test suite (`pytest tests/`): `test_streaks.py` went from 4 passed / 1 failed to 5 passed / 0 failed, and the two unrelated, pre-existing failures in `test_playlists.py` (Issue #5, not part of this fix) are untouched, confirming this change didn't affect anything outside the streak logic. I also reran the original Saturday-to-Sunday reproduction script and confirmed the streak now goes from 1 to 2 instead of resetting to 1.

### Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it:** covered in Milestone 2 above. Simulated a friend listening at 11pm one night, then mocked the clock to check the feed at 9am the next morning, only 10 hours later but a different calendar day. The friend still showed up as "listening now."

**How I found the root cause:** the function's own docstring says it returns friends who listened "recently," and there's a module-level constant, `RECENT_THRESHOLD = timedelta(hours=24)`, used to build the cutoff. That word "recently" is the mismatch: the bug report wants "today" (a calendar concept), but the code was built around a rolling time window (a duration concept). Once I framed it that way, the fix location was obvious, it's the one line that builds `cutoff`.

**The root cause:** in `services/feed_service.py`, `get_friends_listening_now()` computed `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`, a plain 24-hour lookback from the current instant. A listen from 11pm the night before falls well within 24 hours of a 9am check the next morning, so it keeps showing up until a full 24 hours have actually elapsed, which can be most of the next day. The intended behavior (only friends who listened today) needs a calendar-day boundary, not a rolling duration.

**My fix and side-effect check:** replaced the rolling cutoff with the start of the current day in UTC: `cutoff = datetime.combine(now.date(), time.min, tzinfo=timezone.utc)`, and removed the now-unused `RECENT_THRESHOLD` constant and `timedelta` import. I left `get_activity_feed()` untouched since its docstring explicitly says it's not filtered by recency at all, that function doesn't use `cutoff` and doesn't share this bug. I reran the original reproduction: a listen from 11pm the prior night, checked at 9am, now returns 0 results (correctly excluded), and as a control, a listen from earlier the same morning still returns 1 result (correctly included). Full test suite still shows the same 2 pre-existing, unrelated `test_playlists.py` failures (Issue #5) and nothing new.

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** covered in Milestone 2 above. Had simone rate darius's song "Golden Hour" through the live app. The rating saved fine, but darius's notification list stayed empty.

**How I found the root cause:** this is the call chain I already traced for the Milestone 1 codebase map, rating a song and adding a song to a playlist both live in `notification_service.py`, and both are supposed to notify the song's original sharer. I compared them line by line. `add_to_playlist()` ends with an `if song.shared_by != added_by_user_id` check that calls `create_notification()`. `rate_song()` has no equivalent block at all, it commits the `Rating` and returns. I also noticed `create_notification()`'s own docstring already lists `'song_rated'` as an example notification type, a sign this was planned but never wired up.

**The root cause:** in `services/notification_service.py`, `rate_song()` never calls `create_notification()`. This isn't a typo or an off-by-one, it's a missing block of logic. Every other "friend interacted with your song" action in this file follows the pattern of write-then-notify, and this one is simply missing that second half.

**My fix and side-effect check:** added a notification step to `rate_song()` right after the commit, mirroring `add_to_playlist()`'s pattern exactly: skip the notification if the rater is the song's own sharer, otherwise call `create_notification()` with type `"song_rated"` and a message naming the rater, the song, and the score. I tested three cases against the live app: a friend rating someone else's song (notification created correctly), rating it again with a different score (a second notification created, since a changed rating is still a new event worth notifying about), and the sharer rating their own song (correctly produces no notification, same as the self-add case in `add_to_playlist()`). Full test suite shows no new failures, same 2 pre-existing `test_playlists.py` failures as before (Issue #5, not yet fixed).
