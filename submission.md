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