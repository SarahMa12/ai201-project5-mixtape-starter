# AI Usage

I used Claude throughout this project mainly to understand the codebase structure and to trace how specific features worked before I went looking for bugs myself.

At the start, I asked Claude to explain how the app was organized, how routes connect to services, how the blueprint registration in `app.py` maps to URL prefixes, and what each file's role was. This gave me a mental model of the codebase before I started reading individual files.

For specific features, I asked Claude to trace full request flows for me. For example, how `POST /songs/<id>/rate` connects from the route in `routes/songs.py` all the way to `notification_service.rate_song()`, and how `GET /users/<id>/streak` works. This told me which functions and directories to actually go look at rather than reading every file blindly.

When investigating Issue #1, I asked Claude what `datetime.weekday()` and `datetime.isoweekday()` return, since I wasn't sure of the exact values for each day of the week. Claude explained that `weekday()` returns 0 for Monday through 6 for Sunday, and that `isoweekday()` returns 1 for Monday through 7 for Sunday. That's what let me confirm that `today.weekday() != 6` was specifically excluding Sunday from the streak increment.

For Issue #3, I asked Claude to explain what the `outerjoin` in `search_service.py` was doing and why it might cause duplicates. Claude walked me through how a join multiplies rows. I then verified it myself by running the raw SQL query on the command line and seeing the duplicate results directly.

One place I had to verify things myself: Claude initially told me the duplicate rows from Issue #3 would show up in the API response, but when I ran the curl command I only got one result. I went back and Claude explained that SQLAlchemy 2.0 deduplicates ORM objects via the identity map, which is why the API response looked correct even though the underlying SQL was wrong. I confirmed the actual duplicate behavior by running the sqlite3 query directly.

---

# Codebase Map

## File Roles

| File | Role |
| ---- | ---- |
| `app.py` | Flask app factory; initializes SQLAlchemy, registers all blueprints with their URL prefixes |
| `models.py` | Defines all 7 SQLAlchemy models and 3 association tables (see Data Model below) |
| `routes/songs.py` | Handles `/songs` endpoints; delegates to `search_service` and `notification_service` and `streak_service` |
| `routes/playlists.py` | Handles `/playlists` endpoints; delegates to `playlist_service` and `notification_service` |
| `routes/users.py` | Handles `/users` endpoints; delegates to `streak_service` and `notification_service` |
| `routes/feed.py` | Handles `/feed` endpoints; delegates to `feed_service` |
| `services/streak_service.py` | Writes and reads `User.listening_streak`; called on every listen event |
| `services/notification_service.py` | Creates `Notification` rows; also owns `rate_song` and `add_to_playlist` logic |
| `services/search_service.py` | Queries `Song` by title/artist; handles song lookup by ID |
| `services/feed_service.py` | Queries `ListeningEvent` for friends' recent activity |
| `services/playlist_service.py` | Creates playlists and retrieves ordered song lists via `playlist_entries` |
| `seed_data.py` | Populates the DB with users, songs, playlists, and events for manual testing |

---

## Data Model

**Models:** `User`, `Song`, `Tag`, `Rating`, `ListeningEvent`, `Playlist`, `Notification`

**Association tables (many-to-many):**
- `friendships` — links `User` ↔ `User`
- `song_tags` — links `Song` ↔ `Tag`
- `playlist_entries` — links `Playlist` ↔ `Song`, stores `position` and `added_by`

**Key relationships:**
- A `User` shares many `Song`s (`Song.shared_by` → `User.id`)
- A `User` has many `Rating`s and `ListeningEvent`s
- A `User` has many `Notification`s
- A `Song` can belong to many `Playlist`s via `playlist_entries`
- `User.listening_streak` and `User.last_listened_at` are updated by `streak_service` on every listen

---

## All Endpoints

### `/songs` (registered in `app.py:35`)

| Method | Path | Route function | Service call |
| ------ | ---- | -------------- | ------------ |
| GET | `/songs/search?q=` | `routes/songs.py:12` | `search_service.search_songs()` |
| GET | `/songs/<song_id>` | `routes/songs.py:20` | `search_service.get_song()` |
| POST | `/songs/<song_id>/rate` | `routes/songs.py:29` | `notification_service.rate_song()` |
| POST | `/songs/<song_id>/listen` | `routes/songs.py:43` | `streak_service.record_listening_event()` |

### `/playlists` (registered in `app.py:36`)

| Method | Path | Route function | Service call |
| ------ | ---- | -------------- | ------------ |
| POST | `/playlists/` | `routes/playlists.py:10` | `playlist_service.create_playlist()` |
| GET | `/playlists/<playlist_id>` | `routes/playlists.py:25` | `playlist_service.get_playlist()` |
| GET | `/playlists/<playlist_id>/songs` | `routes/playlists.py:34` | `playlist_service.get_playlist_songs()` |
| POST | `/playlists/<playlist_id>/songs` | `routes/playlists.py:43` | `notification_service.add_to_playlist()` |

### `/users` (registered in `app.py:37`)

| Method | Path | Route function | Service call |
| ------ | ---- | -------------- | ------------ |
| GET | `/users/<user_id>` | `routes/users.py:12` | direct `db.session.get(User, ...)` |
| GET | `/users/<user_id>/streak` | `routes/users.py:20` | `streak_service.get_streak()` |
| GET | `/users/<user_id>/notifications` | `routes/users.py:29` | `notification_service.get_notifications()` |
| POST | `/users/notifications/<notification_id>/read` | `routes/users.py:39` | `notification_service.mark_as_read()` |

### `/feed` (registered in `app.py:38`)

| Method | Path | Route function | Service call |
| ------ | ---- | -------------- | ------------ |
| GET | `/feed/<user_id>/listening-now` | `routes/feed.py:9` | `feed_service.get_friends_listening_now()` |
| GET | `/feed/<user_id>/activity` | `routes/feed.py:18` | `feed_service.get_activity_feed()` |

---

## Data Flow: A Friend Adds Your Song to a Playlist

`POST /playlists/<playlist_id>/songs` with `{ song_id, added_by }`

1. **`routes/playlists.py:43`** — parses `song_id` and `added_by` from the request body, then calls `notification_service.add_to_playlist(playlist_id, song_id, added_by_user_id)`
2. **`notification_service.add_to_playlist()` (line 35)** — looks up the `Song`, `User`, and `Playlist` from the DB. If the song is not already in the playlist, appends it: `playlist.songs.append(song)` via the `playlist_entries` association table.
3. **Still in `add_to_playlist()`** — checks whether the person adding the song is the same as the person who originally shared it (`song.shared_by != added_by_user_id`). If they're different, calls `create_notification()`.
4. **`notification_service.create_notification()` (line 13)** — creates a `Notification` row with `user_id` set to the original sharer, `notification_type="song_added_to_playlist"`, and a human-readable `body` string. Commits to DB.
5. **Route returns** `{"message": "Song added to playlist"}` with status 201.

The sharer can then call `GET /users/<id>/notifications` and see the new row.

**Key detail:** `notification_service` owns both the playlist-add logic *and* the notification creation. This is because the service is organized around the *event that triggers notifications*, not around the resource being modified. Adding a song to a playlist lives here rather than in `playlist_service` because its main side effect is a notification.

---

## Patterns

**Routes are thin wrappers.** Every route function does the same three things: parse the request, call exactly one service function, return JSON. No business logic lives in routes. This means if an endpoint is broken, the route file is almost never the cause — trace to the service.

**Services are grouped by the notification they produce, not the model they touch.** `notification_service` owns both `rate_song` and `add_to_playlist` even though those modify `Rating` and `playlist_entries` respectively. The grouping logic is "these actions notify someone," not "these actions touch the same table."

**Streaks are pre-computed, not derived.** `User` stores `listening_streak` and `last_listened_at` as plain columns. Every `POST /songs/<id>/listen` call updates those fields immediately via `update_listening_streak()`. `GET /users/<id>/streak` just reads the stored integer — it does not walk `ListeningEvent` history. This means a bug in the write path won't show up until the next listen event.

**`playlist_entries` is a richer join table.** Unlike `friendships` and `song_tags` which are bare many-to-many links, `playlist_entries` stores `position`, `added_by`, and `added_at`. Songs in a playlist have an explicit order — they are not just a set.

---

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** The bug only triggers on Sundays, so I couldn't hit it by running the app on a Tuesday. Instead I ran the existing test that exercises the Sunday code path:

```bash
pytest tests/test_streaks.py::test_streak_increments_on_sunday -v
```

The test listens on Saturday (streak → 1), then listens again on Sunday and expects the streak to become 2. The test fails — streak stays at 1.

**How I found the root cause:** I opened `services/streak_service.py` and read through `update_listening_streak()`. The function computes `days_since_last` and then branches on it. I traced through the Saturday → Sunday scenario line by line: `days_since_last` is 1, so I looked at the `elif` branch that handles consecutive days. That condition was `days_since_last == 1 and today.weekday() != 6`. On Sunday, `today.weekday()` returns 6, so `!= 6` is False, and the whole condition fails. Execution falls to the `else` branch which resets the streak to 1. That was the specific moment I was confident I'd found it.

**The root cause:** Python's `datetime.weekday()` returns 6 for Sunday. The streak increment condition in `streak_service.py:73` includes `and today.weekday() != 6`, which means any listen event on a Sunday is excluded from the increment branch and falls through to the reset branch instead. There is no rule in the app's requirements that makes Sunday different from any other day — the condition has no valid reason to be there.

**The fix and side-effect check:** Removed `and today.weekday() != 6` from the `elif` condition, leaving `elif days_since_last == 1:`. This means consecutive-day listens increment the streak regardless of which day of the week it is. I ran the full test suite after:

```bash
pytest tests/test_streaks.py -v
```

All tests pass, including the ones for same-day listens, skipped days, and new users — confirming the fix didn't break any other streak behavior.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:** I tested the query used in `search_service.py` on my command line to see if it returned the correct songs:

```bash
sqlite3 instance/mixtape.db "
SELECT song.title FROM song
LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
WHERE song.title LIKE '%Anthem%';"
```

The output showed `Crown Heights Anthem` three times instead of once — the query was returning duplicate rows for songs with multiple tags.

**How I found the root cause:** I opened `services/search_service.py` and read the `search_songs()` function. The query joins `song_tags` onto `Song` before filtering. I looked at the filter — it only checks `Song.title` and `Song.artist`, neither of which are columns in `song_tags`. The join adds nothing to the search logic but multiplies rows for every tag a song has. I confirmed this was the cause by running the raw SQL directly on the database and seeing the duplicate rows.

**The root cause:** `search_service.py:27` does a `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` before filtering by title and artist. A left outer join with `song_tags` produces one row per tag for each matching song. "Crown Heights Anthem" has 3 tags, so it produces 3 rows. The join is not used anywhere in the filter or the return value — `song.to_dict()` loads tags via the `Song.tags` relationship separately — so the join serves no purpose and only causes duplicates.

**The fix and side-effect check:** Removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line entirely. Tags still appear correctly in the response because `Song.to_dict()` loads them through the SQLAlchemy relationship (`Song.tags`), which is independent of this query. I confirmed by searching for "Anthem" and getting one result with all three tags still present in the `tags` field.

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** Got a playlist ID from the database, then requested its songs:

```bash
sqlite3 instance/mixtape.db "SELECT id, name FROM playlist;"
curl http://127.0.0.1:5000/playlists/<id>/songs
```

The seed data puts 7 songs in "Late Night Vibes". The response showed `"count": 6` — the last song was missing.

**How I found the root cause:** I opened `services/playlist_service.py` and read `get_playlist_songs()`. The function queries songs ordered by position and then returns them. The return statement was `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice immediately stood out — that's a Python slice that drops the last element of any list.

**The root cause:** `playlist_service.py:66` uses `songs[:-1]` when building the return list. In Python, `[:-1]` means "everything except the last item." This drops the final song from every playlist response unconditionally, regardless of playlist size. A playlist with 7 songs returns 6; a playlist with 1 song returns 0.

**The fix and side-effect check:** Changed `songs[:-1]` to `songs` so the full list is returned. I re-ran the curl request and got `"count": 7` with all songs present including the last one. I also checked that the songs are still returned in the correct position order, which they are since the query's `order_by(asc(playlist_entries.c.position))` is unchanged.
