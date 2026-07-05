# Mixtape — Project 5 Submission

## Codebase Map

### Main Files

**[app.py](app.py)**
The Flask application factory. `create_app()` configures the database connection (SQLite by default, overridable via `DATABASE_URL`), initializes SQLAlchemy, and registers the four route blueprints with their URL prefixes. It is the entry point that wires everything together. Nothing here does business logic — it just sets up the plumbing.

**[models.py](models.py)**
Defines every database table as a SQLAlchemy model class. There are six models:

- `User` — stores username, email, listening streak count, and last listened timestamp. Has a self-referential many-to-many relationship for friends (via the `friendships` join table).
- `Song` — stores title, artist, album, genre, who shared it, and an optional share note. Songs can have many `Tag`s (via `song_tags` join table).
- `Tag` — a simple label that can be attached to many songs.
- `Playlist` — a named collection with an `is_collaborative` flag. Songs are linked through the `playlist_entries` join table, which also stores position, who added the song, and when.
- `ListeningEvent` — a record that user X listened to song Y at a given time. This is what powers streaks and the activity feed.
- `Rating` — a 1–5 score from a user for a song. Enforces a unique constraint per (user, song) pair so a user can only have one rating per song.
- `Notification` — an in-app alert for a user, with a type string, body text, and a `read` boolean flag.

**[seed_data.py](seed_data.py)**
Populates the database with test users, songs, playlists, friendships, and listening events so the app can be tested without manual setup.

---

### Routes Layer (`routes/`)

The route files are intentionally thin. Each one handles only two things: parsing the incoming HTTP request (extracting JSON body fields or query params) and formatting the outgoing JSON response. All real logic is delegated immediately to a service function. This is a consistent pattern across every route in the app.

- **[routes/songs.py](routes/songs.py)** — four endpoints: search songs, get song detail, rate a song (POST), and log a listening event (POST).
- **[routes/playlists.py](routes/playlists.py)** — four endpoints: create playlist, get playlist metadata, get playlist songs, add a song to a playlist.
- **[routes/users.py](routes/users.py)** — four endpoints: get user profile, get listening streak, get notifications (with optional `?unread_only=true`), mark notification as read.
- **[routes/feed.py](routes/feed.py)** — two endpoints: friends listening now, general activity feed.

---

### Services Layer (`services/`)

This is where all business logic lives. Each service file owns one domain.

**[services/streak_service.py](services/streak_service.py)**
Owns the listening streak feature. `record_listening_event()` creates a `ListeningEvent` row and calls `update_listening_streak()`. That function compares today's date against the user's `last_listened_at` field: same day → no change; 1 day gap → increment; more than 1 day → reset to 1. Also exposes `get_streak()` which just reads the stored value.

**[services/search_service.py](services/search_service.py)**
`search_songs()` runs a SQL `LIKE` query against `Song.title` and `Song.artist` (case-insensitive). Also has `get_song()` to fetch a single song by ID.

**[services/notification_service.py](services/notification_service.py)**
Handles two write operations and two read operations. `add_to_playlist()` appends a song to a playlist and — if the adder is not the song's original sharer — creates a `Notification` for that sharer. `rate_song()` saves or updates a `Rating`. `get_notifications()` retrieves a user's alerts, newest first. `mark_as_read()` flips the `read` flag.

**[services/playlist_service.py](services/playlist_service.py)**
`create_playlist()` creates the playlist record. `get_playlist_songs()` queries songs joined through `playlist_entries` ordered by position. `get_playlist()` returns metadata only. `get_user_playlists()` returns all playlists a user created.

**[services/feed_service.py](services/feed_service.py)**
`get_friends_listening_now()` finds all friends' `ListeningEvent` rows within a recency window, deduplicates to one entry per friend (their most recent), and returns a list of `{friend, song, listened_at}` dicts. `get_activity_feed()` does the same without the recency filter, returning the latest 20 events across all friends.

---

## Data Flow Trace: User rates a song → sharer gets notified

This is the trace the README describes, and it illustrates how the app is organized end-to-end.

```
1. Client sends:  POST /songs/<song_id>/rate
                  Body: { "user_id": "abc", "score": 4 }

2. routes/songs.py → rate()
   - Reads user_id and score from the JSON body
   - Validates both fields are present
   - Calls:  notification_service.rate_song(user_id, song_id, score)

3. services/notification_service.py → rate_song()
   - Validates score is between 1 and 5
   - Confirms the Song exists in the DB
   - Confirms the User exists in the DB
   - Checks if a Rating row already exists for this (user, song) pair
       → If yes: updates the existing score
       → If no:  creates a new Rating row
   - Commits to the database
   - Returns the Rating object

4. routes/songs.py
   - Calls rating.to_dict() to serialize the result
   - Returns HTTP 201 with the rating JSON
```

Note: currently `rate_song()` does NOT notify the song's original sharer — that is one of the five open bugs (Issue #4).

---

## Patterns I Noticed

**Routes delegate immediately.** Every route function does only two things: validate inputs and call a service. No route contains SQL queries or business decisions. This makes the services testable independently of HTTP.

**Services own their domain but sometimes cross into each other.** `notification_service.add_to_playlist()` imports from `playlist_service` to do the actual playlist mutation, then adds the notification on top. This creates a mild coupling: the notification service is doing two jobs (adding songs AND notifying).

**Models are the only persistence layer.** Nothing in routes or services writes raw SQL — everything goes through the SQLAlchemy model classes and `db.session`.

**UUIDs as primary keys.** Every model uses a string UUID as its primary key (generated by `generate_uuid()`), not an auto-incrementing integer. This is common in distributed systems to avoid collisions.

**`to_dict()` on every model.** All models have a `to_dict()` method that serializes the row to a plain Python dict. Routes call this before calling `jsonify()`. Tags are embedded directly into the song dict rather than being a separate API response.

---

## The Five Open Issues

| # | Issue | File | Bug |
|---|-------|------|-----|
| 1 | Streak keeps resetting | [services/streak_service.py](services/streak_service.py) | Line 73 has `and today.weekday() != 6` — this prevents the streak from incrementing on Sundays, resetting it to 1 instead. |
| 2 | Listening Now shows people from yesterday | [services/feed_service.py](services/feed_service.py) | `RECENT_THRESHOLD = timedelta(hours=24)` is too wide for "listening now" — someone from 23 hours ago is not listening now. |
| 3 | Same song appears twice in search | [services/search_service.py](services/search_service.py) | The `outerjoin` on `song_tags` multiplies rows — a song with 2 tags appears twice in results. Needs `.distinct()`. |
| 4 | No notification when a friend rates my song | [services/notification_service.py](services/notification_service.py) | `rate_song()` never calls `create_notification()`. The notification is only sent for playlist adds, not ratings. |
| 5 | Last song in a playlist never shows up | [services/playlist_service.py](services/playlist_service.py) | Line 66: `return [song.to_dict() for song in songs[:-1]]` — `[:-1]` slices off the last item. Should be `songs` (no slice). |

---

## Bug Reports (Issues 1, 3, 5)

### Issue 1 — Streak resets on Sundays

**How I reproduced it:**

Ran the test that simulates a Saturday → Sunday consecutive listening sequence using fixed hardcoded dates:

```
pytest tests/test_streaks.py::test_streak_increments_on_sunday -v
```

The test calls `update_listening_streak()` twice: first with `datetime(2024, 6, 15, tzinfo=UTC)` (a Saturday, `weekday() == 5`) to set the baseline, then with `datetime(2024, 6, 16, tzinfo=UTC)` (the following Sunday, `weekday() == 6`). Because only one calendar day separates the two events, the streak should increment from 1 to 2. Instead the function returned a streak of 1 — the Sunday listen had reset it. The test uses fixed dates so the failure is deterministic and reproducible on any day of the week.

Observed failure output:
```
assert 1 == 2
 +  where 1 = <User ...>.listening_streak
```

**How I found the root cause:**

Started in [routes/songs.py](routes/songs.py) at the `POST /<song_id>/listen` endpoint, which calls `record_listening_event()` in [services/streak_service.py](services/streak_service.py). That function creates the `ListeningEvent` row and then delegates to `update_listening_streak()` — the function the test calls directly. I read the three-branch `if/elif/else` block in `update_listening_streak()` top to bottom. The first branch handles a first-ever listen, the `else` resets the streak, and the `elif` is the only branch that increments. The moment I read the `elif` condition — `days_since_last == 1 and today.weekday() != 6` — the cause was unambiguous: any Sunday would fail the `!= 6` check and skip the increment entirely, regardless of whether the days were actually consecutive.

**The root cause:**

Python's `date.weekday()` returns an integer where Monday = 0 and Sunday = 6. The `elif` condition in [streak_service.py:73](services/streak_service.py#L73) was:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

The `today.weekday() != 6` guard evaluates to `False` whenever today is Sunday. This makes the entire `elif` False on Sundays — even when `days_since_last == 1` is True, meaning the user genuinely did listen on consecutive days (Saturday then Sunday). Because the `elif` is skipped, execution falls through to the `else` branch, which unconditionally resets `listening_streak` to 1. The streak is not broken — it is erased by the wrong branch being taken.

**Fix and side-effect check:**

Removed `and today.weekday() != 6` from the `elif` condition, leaving:

```python
elif days_since_last == 1:
```

`days_since_last == 1` is already the complete, correct test for consecutive calendar days. No weekday guard is needed or appropriate — a Saturday-to-Sunday gap is exactly one day, the same as any other consecutive pair. The removed condition added nothing correct and subtracted valid increments every week.

After the fix, ran the complete streak test suite to check all boundaries:

```
pytest tests/test_streaks.py -v   →   5 passed
```

Boundary cases covered by the existing tests:
- First-ever listen → streak starts at 1 ✓
- Same day, second listen → streak does not double-count ✓
- Consecutive days (Mon → Tue) → streak increments ✓
- Skipped day (Mon → Wed) → streak resets ✓
- Saturday → Sunday → streak now correctly increments ✓ (was the failing case)

Ran the full suite afterward to confirm no regressions in search or playlist tests.

---

### Issue 3 — Same song appears twice in search

**How I reproduced it:**
Ran the test that searches for a song that has three tags assigned to it:

```
pytest tests/test_search.py::test_search_no_duplicates_multi_tag_song -v
```

**Note:** This test currently passes. The reason is that SQLAlchemy's identity map collapses duplicate rows into a single Python object when querying a full model — so the ORM hides the problem. The duplicate rows exist in the SQL output but are deduplicated before `.all()` returns them. The underlying query is still incorrect: inspecting the SQL shows one row per (song, tag) pair, meaning a song with 3 tags produces 3 rows. The fix (removing the unnecessary join) is still correct and makes the query unambiguously safe.

**Required data condition:** The bug manifests only for songs that have 2 or more tags. A song with 0 or 1 tags is not affected because there are no duplicate join rows to collapse.

**Root cause:** [search_service.py:27](services/search_service.py#L27) — the `outerjoin(song_tags, ...)` is not needed. Tags are already eagerly loaded via the `song.tags` relationship defined on the `Song` model and used inside `to_dict()`. The join multiplies rows without adding any information that isn't already available.

---

### Issue 5 — Last song in a playlist never shows up

**How I reproduced it:**
Ran the playlist tests against a playlist seeded with 5 songs (Track 1 through Track 5):

```
pytest tests/test_playlists.py -v
```

**Required data condition:** Any playlist with at least 2 songs. The last song (highest position value) is always the one dropped, regardless of how many songs are in the playlist.

**Observed failures:**
```
test_playlist_returns_all_songs     FAILED — assert 4 == 5
test_playlist_returns_songs_in_order FAILED — list ends at 'Track 4', missing 'Track 5'
```

**Root cause:** [playlist_service.py:66](services/playlist_service.py#L66) — `songs[:-1]` is Python slice notation meaning "everything except the last element." The query fetches all songs correctly; the slice then removes the final one before returning. Changing `songs[:-1]` to `songs` fixes it.
