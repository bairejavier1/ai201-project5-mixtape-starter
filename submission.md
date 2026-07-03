# Mixtape — Submission

## Codebase Map

**Main files:**
- `app.py` — Flask app factory, registers 4 blueprints (songs, playlists, users, feed),
  initializes the SQLAlchemy db object.
- `models.py` — 6 models: User, Tag, Song, ListeningEvent, Rating, Playlist,
  Notification, plus 3 association tables (friendships, song_tags, playlist_entries).
  Playlist membership is a many-to-many table with an explicit `position` column, not
  insertion order.
- `routes/` — one blueprint per resource (songs, playlists, users, feed). Every route
  does input parsing + calls a service function + formats the JSON response. No business
  logic lives in routes.
- `services/` — all business logic. streak_service (listening streaks),
  feed_service (friends' recent activity), search_service (song search),
  notification_service (creating/reading notifications, plus rating logic oddly lives
  here too), playlist_service (playlist retrieval/creation).

**Data flow — a user rates a song:**
`POST /songs/<id>/rate` (routes/songs.py) → `notification_service.rate_song()` →
creates or updates a `Rating` row (unique per user+song via a DB constraint) → commits.
Notably, this path does NOT call `create_notification()`, unlike the sibling function
`add_to_playlist()` in the same file, which does notify the original sharer.

**Data flow — a user adds a song to a playlist:**
`POST /playlists/<id>/songs` → `notification_service.add_to_playlist()` → appends the
song to `playlist.songs` (the many-to-many relationship) → commits → if the adder isn't
the original sharer, creates a Notification for the sharer.

**Pattern noticed:** routes are thin, services hold all logic. Interestingly, rating logic
lives in `notification_service.py` rather than its own file — a hint that notification
side-effects were meant to be attached to it but weren't finished for ratings.

## Root Cause Analysis

### Issue #1: My listening streak keeps resetting

**How I reproduced it:** I wrote a small script that called `update_listening_streak()`
directly with two consecutive dates — a Saturday and the following Sunday
(2024-06-15 and 2024-06-16, both UTC). After the Saturday call the streak was 1 as
expected, but after the Sunday call it stayed at 1 instead of incrementing to 2, even
though the two dates were one day apart.

**How I found the root cause:** I opened `services/streak_service.py` and read
`update_listening_streak()` top to bottom. The function branches on `days_since_last`:
0 (no change), 1 (increment), anything else (reset to 1). I noticed the `days_since_last == 1`
branch had an extra condition, `and today.weekday() != 6`. I confirmed `datetime.weekday()`
returns 6 for Sunday, which meant the increment branch was being skipped specifically on
Sundays, sending execution to the `else` reset branch instead.

**The root cause:** The condition `today.weekday() != 6` excludes Sunday from the
streak-increment branch. Since Python's `weekday()` returns 6 for Sunday, any listen
that happens on a Sunday after exactly a 1-day gap fails this condition and falls
through to the `else` clause, which resets the streak to 1 — even though the user
listened on consecutive days and the streak should have incremented.

**My fix and side-effect check:** I removed the `and today.weekday() != 6` clause,
leaving `elif days_since_last == 1:` to increment the streak on any 1-day gap regardless
of the day of the week. I re-ran the full `tests/test_streaks.py` suite (5 tests) to
confirm same-day handling, skipped-day resets, and normal weekday increments were
unaffected — all 5 passed, including the previously-failing Sunday test.

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:** I ran `pytest tests/test_playlists.py -v`. Two tests failed:
`test_playlist_returns_all_songs` expected 5 songs back but got 4, and
`test_playlist_returns_songs_in_order` showed the returned list was missing "Track 5"
specifically — the last song by position — while the first four tracks were present
and correctly ordered.

**How I found the root cause:** I opened `services/playlist_service.py` and read
`get_playlist_songs()`. The query itself builds the song list correctly, ordered
ascending by `position`. The very last line of the function is
`return [song.to_dict() for song in songs[:-1]]`. Seeing `[:-1]` applied to an
already-correct, already-ordered list immediately explained why exactly one song —
always the last one — was missing regardless of playlist length, which matched the
test failures precisely.

**The root cause:** The list comprehension slices the ordered `songs` list with
`songs[:-1]`, which drops the final element before serializing to dicts. This has
nothing to do with the SQL query or ordering logic — both are correct — it's a plain
Python slicing mistake applied after the correct list was already built, so it silently
discards the last song in every playlist regardless of size.

**My fix and side-effect check:** I changed `songs[:-1]` to `songs`, removing the slice
entirely so the full ordered list is returned. I re-ran `tests/test_playlists.py` (all
3 tests, including the empty-playlist edge case) to confirm the fix and that an empty
playlist still correctly returns `[]` rather than erroring on the removed slice. I also
ran the full `pytest tests/` suite to confirm this change didn't affect the search or
streak tests, since `playlist_service.py` isn't imported by either.

