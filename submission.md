# Mixtape Bug Hunt — Submission

## AI Usage

*(To be completed in Milestone 4, after all bugs are fixed — this section will describe specifically how AI was used to navigate and debug, and where its output was verified or overridden.)*

## Codebase Map

### Main files and their roles

**`app.py`** — Flask application factory (`create_app`). Owns the single `SQLAlchemy` `db` instance, sets the SQLite URI, registers the four route blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and calls `db.create_all()` on startup.

**`models.py`** — Defines all 7 SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables:
- `friendships` — many-to-many `User`↔`User`, inserted as two rows per friendship (one each direction) to make it symmetric.
- `song_tags` — plain many-to-many `Song`↔`Tag`.
- `playlist_entries` — many-to-many `Playlist`↔`Song`, but *enriched* with extra columns (`position`, `added_by`, `added_at`). Song order in a playlist is an explicit stored integer, not just insertion/row order — this matters for Issue #5.

**`routes/`** — Thin controllers. Every route function parses query args or the JSON body, calls exactly one function in `services/`, serializes the result with `.to_dict()`, and translates a raised `ValueError` into a 4xx JSON response. No business logic lives here — this is a consistent convention across all four route files.
- `songs.py`: `/songs/search` → `search_service.search_songs`; `/songs/<id>` → `search_service.get_song`; `/songs/<id>/rate` (POST) → `notification_service.rate_song`; `/songs/<id>/listen` (POST) → `streak_service.record_listening_event`.
- `playlists.py`: create playlist, `/playlists/<id>`, `/playlists/<id>/songs` (GET) → `playlist_service.get_playlist_songs`, `/playlists/<id>/songs` (POST) → `notification_service.add_to_playlist`.
- `users.py`: `/users/<id>`, `/users/<id>/streak` → `streak_service.get_streak`, `/users/<id>/notifications` → `notification_service.get_notifications`, mark-as-read.
- `feed.py`: `/feed/<id>/listening-now` → `feed_service.get_friends_listening_now`, `/feed/<id>/activity` → `feed_service.get_activity_feed`.

**`services/`** — All business logic lives here.
- `streak_service.py` — `record_listening_event` (creates a `ListeningEvent`, then calls `update_listening_streak`), `update_listening_streak` (the day-over-day streak math), `get_streak`.
- `feed_service.py` — `get_friends_listening_now` (friends' most recent listen, filtered to a recency window) and `get_activity_feed` (last N events from friends, no time filter — deliberately different semantics from the first function).
- `search_service.py` — `search_songs` (joins `Song` to `song_tags` and filters by `ilike` on title/artist), `get_song`.
- `notification_service.py` — `create_notification` (generic writer), `add_to_playlist` (adds a song to a playlist *and* notifies the original sharer), `rate_song` (saves a `Rating`), `get_notifications`, `mark_as_read`.
- `playlist_service.py` — `create_playlist`, `get_playlist_songs` (orders songs by `position`), `get_playlist`, `get_user_playlists`.

**`seed_data.py`** — Populates 5 users with a friendship graph, 13 songs (deliberately split into 0-tag / 1-tag / 3-tag groups), 3 playlists, a mix of very-recent and multi-day-old listening events, and one pre-existing "song added to playlist" notification so the working notification pattern is visible for comparison against Issue #4.

**`tests/`** — `test_streaks.py` and `test_playlists.py` already contain failing assertions that pin down Issues #1 and #5 exactly (confirmed by running `pytest tests/` before touching any code — see reproduction notes below). `test_search.py` currently passes in full, which will need to be reconciled with Issue #3 during reproduction.

### Data flow — rating a song (and why it doesn't notify)

`POST /songs/<song_id>/rate` with `{user_id, score}` → `routes/songs.py::rate()` parses the body and calls `notification_service.rate_song(user_id, song_id, score)`. That function validates the score, looks up the song and rater, checks for an existing `Rating` (there's a `UniqueConstraint` on `user_id`+`song_id`), creates-or-updates the row, commits, and returns the `Rating`. **It never calls `create_notification`.**

Compare this to the sibling flow: `POST /playlists/<id>/songs` → `routes/playlists.py::add_song()` → `notification_service.add_to_playlist(playlist_id, song_id, added_by)`. That function does the same "save the record" work, but then has an extra block: `if song.shared_by != added_by_user_id: create_notification(...)`, notifying whoever originally shared the song. `rate_song` has no equivalent block — this asymmetry is the root of Issue #4 and is visible just from reading the two functions side by side in the same file.

### Data flow — reading a playlist's songs (and why the last one goes missing)

`GET /playlists/<id>/songs` → `routes/playlists.py::get_songs()` → `playlist_service.get_playlist_songs(playlist_id)`. This joins `Song` to `playlist_entries` on `playlist_id`, orders ascending by `position`, executes, converts each result to a dict — **and then returns `songs[:-1]`**, silently dropping the last element of the ordered list before sending it back. Since `position` increases with each added song, the last element after `ORDER BY position ASC` is always the most recently added song — matching Issue #5's report exactly ("the missing one is always whatever was added most recently").

### Patterns noticed

- Every route is a pure "parse → call one service → serialize" controller; there is no business logic in `routes/`.
- `playlist_entries` stores explicit order (`position`) rather than relying on row/insertion order — a deliberate modeling choice.
- Services consistently raise `ValueError` for not-found/invalid-input cases, and every route consistently catches it and returns the matching 4xx — this convention holds across all four route modules without exception.
- `feed_service.py` has two similarly-named functions with different filtering semantics (`get_friends_listening_now` is time-windowed and deduped per friend; `get_activity_feed` is just "last N, no time filter"). Issue #2 lives specifically in the first one — worth not conflating the two while investigating.

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** Ran `pytest tests/` before touching any code. `test_streak_increments_on_sunday` in `tests/test_streaks.py` failed: it calls `update_listening_streak(user, saturday)` (streak → 1), then `update_listening_streak(user, sunday)` — a consecutive calendar day — and expects the streak to go to 2. Instead it stayed at 1. This matches kenji's report exactly: listening on consecutive days, with the reset specifically happening when the second day is a Sunday.

**How I found the root cause:** `services/streak_service.py` is the file the README maps to this issue, so I started there. `update_listening_streak()` is a ~35-line function with one conditional block deciding streak behavior based on `days_since_last`. Reading it top to bottom, the branch handling the "listened on a consecutive day" case immediately stood out: `elif days_since_last == 1 and today.weekday() != 6:`. That extra `and today.weekday() != 6` clause has no basis in the function's own docstring ("If the user listened yesterday: streak increments by 1" — no day-of-week exception is mentioned), which is what made me confident this was the actual defect and not just a suspicious area.

**The root cause:** Python's `datetime.weekday()` returns `0` for Monday through `6` for Sunday. The condition `today.weekday() != 6` is meant to exclude something, but as written it excludes Sundays specifically from the streak-increment branch. So whenever a user listens on consecutive calendar days and the second listen happens to fall on a Sunday, `days_since_last == 1` is `True` but the `weekday() != 6` check is `False`, so the `elif` fails and execution falls through to the `else` branch, which unconditionally resets `listening_streak` to `1` — treating a legitimate consecutive-day listen exactly like a skipped day.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` condition entirely, so the branch is simply `elif days_since_last == 1:` — any consecutive-day listen increments the streak, regardless of which day of the week it lands on. I checked the other two branches (`days_since_last == 0` → no change, and the final `else` → reset to 1) are untouched and still correct for the "already listened today" and "skipped a day" cases. I also re-read `record_listening_event()` (the only caller) to confirm it always passes the true current UTC time, so no other caller depends on the old Sunday-specific behavior. Ran `pytest tests/test_streaks.py -v` after the fix — all streak tests pass, including the one that previously failed.

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** Ran `pytest tests/` before touching any code (same run that caught Issue #1). `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` in `tests/test_playlists.py` both failed: a 5-song seeded playlist came back with only 4 songs, and the missing one was always the last in position order ("Track 5"). This matches darius's report exactly — a playlist that should show 7 songs shows 6, and the missing one is always the most recently added.

**How I found the root cause:** The README maps this issue to `services/playlist_service.py`. `get_playlist_songs()` is short and its docstring even explicitly states "This function returns all songs in the playlist" — a strong signal something contradicts that claim. The query itself (join to `playlist_entries`, filter by playlist, order by `position` ascending) is correct and matches the docstring's description of ordering. The very last line is where the return value is actually built: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice is the only place in the function that could drop a row, and it does so unconditionally on an otherwise-correct, correctly-ordered list — that's what made me confident this was the actual bug rather than something upstream in the query.

**The root cause:** `songs` is already the fully correct, ordered list of every song in the playlist (ordered ascending by `position`, so index `0` is the first song added and the last index is the most recently added). The line `songs[:-1]` slices off the final element of that list before converting to dicts and returning. Since `position` increases monotonically as songs are added, the last element after `ORDER BY position ASC` is always the most recently added song — so this slice doesn't drop a random song, it deterministically drops whichever song was added last. That's why darius saw the playlist "hide exactly one song: the last one added," and why adding a new song made the previously-missing song reappear while the new one became the new "last" (and thus newly hidden) one.

**My fix and side-effect check:** Removed the `[:-1]` slice so the function returns `[song.to_dict() for song in songs]` — the full, already-correctly-ordered list, matching the function's own docstring. I checked `get_playlist()` (the sibling function that returns playlist metadata without songs) and `get_user_playlists()` — neither touches `playlist_entries` or slices a song list, so they were never affected by this bug and don't need changes. I also checked `notification_service.add_to_playlist()`, which calls `playlist.songs` (the SQLAlchemy relationship, not `get_playlist_songs()`) to check membership before appending — a different code path entirely, unaffected by this fix. Ran `pytest tests/test_playlists.py -v` after the fix — both previously-failing tests now pass.

*(More entries added as remaining bugs are fixed.)*
