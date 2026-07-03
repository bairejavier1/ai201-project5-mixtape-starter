## AI Usage

I used Claude throughout this project for codebase navigation and debugging support,
not for blindly generating fixes.

- **Codebase orientation:** I asked Claude to read through `models.py`,
  `notification_service.py`, and `feed_service.py` and explain what each function was
  responsible for and how routes connected to services, before I wrote my codebase map.
- **Root cause hypotheses:** For each bug, Claude proposed a specific root cause based
  on reading the code (e.g., the `weekday() != 6` condition for Issue #1, the `[:-1]`
  slice for Issue #5, the missing `create_notification()` call for Issue #4, and the
  24-hour threshold for Issue #2). I did not accept any of these on faith — for each
  one, I wrote and ran a reproduction script myself to confirm the bug actually behaved
  the way Claude predicted before writing any fix.
- **Where I had to course-correct the AI:** Claude predicted Issue #3 (duplicate search
  results from a many-to-many join) would reproduce based on reading the code, but when
  I actually ran `pytest tests/test_search.py`, all 5 tests passed — the bug didn't
  reproduce in my environment (likely due to my SQLAlchemy version handling the join
  differently than expected). Rather than force a fix for something that wasn't actually
  broken, I dropped that issue and worked on Issue #2 instead. This is a case where the
  AI's code-reading analysis was reasonable but incomplete without actually running the
  code, and I had to verify it independently.
- **Fix verification:** For every fix, I re-ran the relevant pytest suite and/or my own
  reproduction scripts myself to confirm the behavior changed before committing. Claude
  suggested the specific code changes, but I verified every one against real output
  before accepting it.
  

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


### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** I wrote a script that created two users (a sharer and a rater)
and one song shared by the sharer, then called `rate_song(rater.id, song.id, 5)` and
checked `get_notifications(sharer.id)` afterward. The result was an empty list —
`Count: 0` — confirming the sharer received no notification even though their song was
rated by someone else.

**How I found the root cause:** I opened `services/notification_service.py` and
compared `rate_song()` against the working `add_to_playlist()` function in the same
file, since the assignment hinted the two should be structurally similar. `add_to_playlist()`
ends with a check (`if song.shared_by != added_by_user_id`) followed by a call to
`create_notification()`. `rate_song()` has no equivalent check or call anywhere in its
body — it saves or updates the `Rating` row, commits, and returns. That absence, not
any incorrect logic, was the root cause.

**The root cause:** `rate_song()` never calls `create_notification()`. Unlike
`add_to_playlist()`, which notifies the original sharer after its main action, the
rating flow has no notification step implemented at all. This is a missing piece of
functionality rather than a broken comparison or off-by-one error — the sharer is
never informed that a friend rated their song, regardless of who rates it or what score
they give.

**My fix and side-effect check:** I added a notification step immediately after the
`db.session.commit()` in `rate_song()`, mirroring the pattern in `add_to_playlist()`:
if the rater isn't the original sharer, call `create_notification()` with a
`"song_rated"` type and a message naming the rater, song, and score. I re-ran my
reproduction script and confirmed the sharer now receives exactly one notification with
the correct body text. I also ran the full `pytest tests/` suite (all 13 tests) to
confirm the playlist and search tests were unaffected, since they don't touch
`rate_song()` at all.


### Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it:** I wrote a script that created two friended users, had one of
them "listen" to a song with a `listened_at` timestamp set 20 hours in the past, and
called `get_friends_listening_now()` for the other user. The friend appeared in the
result with `Count: 1`, even though they hadn't listened to anything in the last 20
hours — confirming stale activity was being shown as current.

**How I found the root cause:** I opened `services/feed_service.py` and read
`get_friends_listening_now()`. The function filters `ListeningEvent` rows using
`listened_at >= cutoff`, where `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`
and `RECENT_THRESHOLD = timedelta(hours=24)`. The filtering logic itself is correct —
it does exactly what it says. The problem is the threshold value itself: a full 24-hour
window is far too generous for a feature meant to show who is listening "now." I
confirmed this by testing the boundary directly: a 20-hour-old event passed the filter
(showed up), while reducing the threshold to 30 minutes correctly excluded it.

**The root cause:** `RECENT_THRESHOLD` was set to `timedelta(hours=24)`, meaning any
listening event from within the last full day counts as "listening now." This isn't a
comparison or logic error — the code correctly implements a 24-hour rolling window — but
a 24-hour window is the wrong product behavior for a "currently listening" feed, since
it surfaces activity from many hours earlier (effectively "yesterday" from the user's
perspective) as if it were happening right now.

**My fix and side-effect check:** I changed `RECENT_THRESHOLD` from `timedelta(hours=24)`
to `timedelta(minutes=30)`. I verified this two ways: re-running my original repro
script confirmed the 20-hour-old event no longer appears (`Count: 0`), and a second
script with a 5-minutes-ago event confirmed genuinely recent activity still appears
correctly (`Count: 1`). I also ran the full `pytest tests/` suite (13 tests) to confirm
the streak, search, and playlist tests were unaffected, since none of them touch
`feed_service.py`.


## Regression Test

`tests/test_streaks.py::test_streak_increments_on_sunday` verifies that listening on a
Saturday and then the following Sunday increments the streak from 1 to 2. Against the
original buggy code (`elif days_since_last == 1 and today.weekday() != 6:`), this test
fails because the Sunday listen incorrectly falls through to the reset branch instead of
incrementing. Against my fix, it passes. This test was already present in the repo
before I started, so it would have caught this bug before it was ever introduced.

`tests/test_playlists.py::test_playlist_returns_all_songs` similarly would have caught
Issue #5 — it asserts a 5-song playlist returns all 5 songs, which fails against the
original `songs[:-1]` slice and passes against my fix.